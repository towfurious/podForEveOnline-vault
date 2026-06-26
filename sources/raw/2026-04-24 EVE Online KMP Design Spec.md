# EVE Online KMP Dashboard — Design Spec

**Date:** 2026-04-24
**Stack:** Kotlin Multiplatform + Compose Multiplatform + Material 3
**Platforms:** Android, iOS
**Scope:** Single character, MVP with scaling headroom

---

## Goal

A glanceable "industrialist smoke-check" app: open it, understand in 5 seconds whether you need to log into the EVE client. Covers skill training, planetary interaction, and research/manufacturing jobs.

---

## Architecture

### Layers

```
┌─────────────────────────────────────┐
│  UI Layer (Compose Multiplatform)   │
│  Screens + ViewModels (shared)      │
├─────────────────────────────────────┤
│  Domain Layer (shared)              │
│  Use Cases, Domain Models           │
├─────────────────────────────────────┤
│  Data Layer (shared)                │
│  Ktor ESI client, SQLDelight cache  │
│  Repositories                       │
├─────────────────────────────────────┤
│  Platform (expect/actual)           │
│  Android: ForegroundService         │
│  iOS: UNUserNotificationCenter      │
│  SecureStorage: Keystore/Keychain   │
└─────────────────────────────────────┘
```

### Module structure

```
/composeApp          — shared UI + ViewModels (Compose Multiplatform)
/shared              — domain + data layer (pure KMP)
  /commonMain
  /androidMain
  /iosMain
/androidApp          — Android entry point, ForegroundService
/iosApp              — iOS entry point, AppDelegate
```

### Key libraries

| Library | Purpose |
|---|---|
| Ktor | ESI HTTP client (KMP-native) |
| SQLDelight | Local cache + schema versioning |
| Koin | Dependency injection (KMP) |
| Voyager | Navigation in Compose Multiplatform |
| Coil 3 | Character portrait loading (KMP) |

---

## Authentication — EVE SSO OAuth2 PKCE

### Flow

1. App generates `code_verifier` (128-bit `SecureRandom`) + `code_challenge` (SHA-256)
2. Opens EVE login URL via system browser:
   - **Android:** Chrome Custom Tabs (`CustomTabsIntent`). Falls back to `Intent.ACTION_VIEW` if no Custom Tabs-compatible browser is installed. Respects user's default browser.
   - **iOS:** `ASWebAuthenticationSession`
3. EVE redirects to `eve-tracker://callback?code=...`
4. App exchanges code → access token + refresh token
5. Tokens stored via `expect class SecureStorage`:
   - **Android:** `EncryptedSharedPreferences` (Android Keystore)
   - **iOS:** Keychain via Security framework
6. Access token (TTL 20 min) kept in memory only, never persisted
7. Refresh token persisted in Keystore/Keychain, never in SQLDelight or logs

### Security details

- PKCE mandatory, no client secret (public client)
- `state` parameter against CSRF
- Redirect URI: `eve-tracker://callback` registered in EVE Developer Portal, `AndroidManifest.xml`, and `Info.plist`

### ESI scopes (MVP)

```
esi-skills.read_skills.v1
esi-skills.read_skillqueue.v1
esi-wallet.read_character_wallets.v1
esi-planets.manage_planets.v1
esi-industry.read_character_jobs.v1
```

---

## Navigation

`NavigationBar` (M3 bottom nav) — 4 items for phones.
Adaptive: `NavigationRail` on tablets / landscape.

```
Dashboard  |  Skills  |  PI  |  Jobs
```

5th slot reserved for future expansion (Assets, Mail, etc.) — no navigation refactor needed to add it.

---

## Screens

### Dashboard

- Character portrait (Coil) + name + corporation
- ISK balance
- Current training skill: name, level, mini progress bar, time remaining
- Summary rows: "N planets need attention", "N jobs completed"
- Skeleton shimmer while loading

### Skills

- Large live progress bar for current skill (calculated from `start_sp`, `finish_sp`, `start_date`, `finish_date` — no polling needed)
- Time to finish current skill / full queue
- Scrollable skill queue list (skill name, level, duration)
- Skeleton shimmer while loading

### PI (Planetary Interaction)

One card per planet:

| Field | Detail |
|---|---|
| Planet name + type | e.g. "Temperate IV" |
| Status chip | Active / Needs Attention / Idle |
| Extractor status | Running / Depleted / time until next cycle |
| Factory chains | OK / Stopped / Starved |

Goal: user reads cards and knows immediately whether to log in.

### Jobs (Research & Industry)

List of active jobs per character:

| Field |
|---|
| Job type (ME Research / TE Research / Manufacturing / Copying) |
| Blueprint name |
| Station / Structure name |
| Progress bar |
| Time remaining |

---

## Notifications & Background

### Android — ForegroundService

- Persistent notification in shade with live skill name + countdown timer
- Updates every second via coroutine, math-only (no API calls during update)
- On skill queue change: re-fetches ESI, reschedules `AlarmManager` for exact `finish_date`
- User can dismiss only by stopping training (swipe = stops service, not possible; service is foreground)

### iOS — Local Notifications

- On queue load: schedules `UNNotificationRequest` for exact `finish_date`
- `BGAppRefreshTask` used for periodic queue sync (best-effort, iOS controls timing)
- No equivalent of Android foreground service — notification fires at scheduled time reliably

---

## UI State Model

All screens use a shared sealed class:

```kotlin
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

`Loading` → shimmer skeleton (Modifier-based, no extra library needed).
`Error` → inline error with retry button.

---

## Core Design System

Thin layer over Material 3 — no custom design system overhead:

- `EveTheme` — wraps `MaterialTheme`, dark theme by default (matches EVE aesthetic), EVE-palette accent colors
- `EveCard` — base card with built-in Loading/Success/Error state rendering
- `ProgressSkillBar` — reusable progress bar used in Skills screen, Dashboard, and Android notification
- `StatusChip` — "Active" / "Needs Attention" / "Completed" / "Idle"

---

## Data Refresh Strategy

- On app foreground: refresh all ESI data
- Pull-to-refresh on each screen
- SQLDelight cache: serve stale data immediately, refresh in background
- Cache TTL matches ESI cache headers

---

## Unit Tests (`commonTest`)

| Test | What |
|---|---|
| `SkillProgressCalculatorTest` | SP math: given start/finish/now → correct percentage and time remaining |
| `EsiResponseParserTest` | ESI JSON → domain models (skills, PI, jobs) |
| `PiStatusUseCaseTest` | Planet status logic: Active / Needs Attention / Idle |
| `OAuthPkceTest` | code_verifier entropy, code_challenge SHA-256 correctness |
| `SkillQueueViewModelTest` | State transitions: Loading → Success → queue update |

---

## Out of Scope (MVP)

- Market data
- Assets (full inventory)
- Corporation / alliance features
- Multiple characters
- Mail / notifications inbox
- Fitting tool

These are non-breaking additions — navigation, data layer, and screen slots are designed to accommodate them.
