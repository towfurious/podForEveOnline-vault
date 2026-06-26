---
title: Source - 2026-04-24 - EVE Online KMP Design Spec
type: source
tags: [source, spec, mvp, architecture]
aliases: [Design Spec, EVE Spec]
created: 2026-04-24
updated: 2026-04-24
status: active
source-path: sources/raw/2026-04-24 EVE Online KMP Design Spec.md
---

# Source — 2026-04-24 — EVE Online KMP Design Spec

## Provenance
Design spec produced during the brainstorming/spec-writing phase of this project. Approved by the user on 2026-04-24. Original lives at `/home/towfuri0us/docs/superpowers/specs/2026-04-24-eve-online-kmp-design.md`; a snapshot lives at `sources/raw/2026-04-24 EVE Online KMP Design Spec.md`. This is the architectural source of truth for the MVP.

## One-page summary
A Kotlin Multiplatform + Compose Multiplatform app for Android and iOS. Single character, MVP. The user (EVE industrialist) opens it and in 5 seconds knows whether to log into the EVE client. Four screens: Dashboard (portrait, ISK, current training, summary), Skills (live progress bar + queue), PI (per-planet cards with Active/Needs Attention/Idle), Jobs (research + manufacturing). Authentication via EVE SSO OAuth2 PKCE through system browser (Chrome Custom Tabs / ASWebAuthenticationSession). Tokens stored in Keystore/Keychain via `expect class SecureStorage`. Data layer: Ktor HTTP client for ESI, SQLDelight stale-while-revalidate cache, Koin DI, Voyager navigation, Coil 3 for character portraits. Notifications asymmetric per platform: Android runs a `ForegroundService` with a persistent countdown notification updated every second via coroutine (math-only, no polling); iOS schedules `UNNotificationRequest` for exact `finish_date` plus `BGAppRefreshTask` for best-effort queue syncs. All screens use a shared `UiState<T>` sealed class (Loading renders Modifier-based shimmer, Error inline with retry). Material 3 dark-by-default theme with EVE-palette accents. Navigation: `NavigationBar` with 4 items on phones, `NavigationRail` on tablets, a 5th slot reserved for future expansion.

## Key facts / claims
- **Stack:** KMP + Compose Multiplatform + Material 3 + Ktor + SQLDelight + Koin + Voyager + Coil 3.
- **Modules:** `/composeApp` (shared UI + VMs), `/shared` (domain + data), `/androidApp` (entry + FG service), `/iosApp` (entry + AppDelegate).
- **Auth:** EVE SSO OAuth2 PKCE (no client secret), `state` for CSRF, redirect `eve-tracker://callback`. Access token in memory only (20-min TTL); refresh token persisted to `SecureStorage` (Keystore / Keychain).
- **ESI scopes (MVP):** `esi-skills.read_skills.v1`, `esi-skills.read_skillqueue.v1`, `esi-wallet.read_character_wallets.v1`, `esi-planets.manage_planets.v1`, `esi-industry.read_character_jobs.v1`.
- **Progress math:** live skill progress computed from `start_sp`, `finish_sp`, `start_date`, `finish_date`. No polling. UI ticks via coroutine; queue-change events re-fetch and reschedule.
- **UI state:** `UiState<T>` sealed class shared across all screens; Loading → shimmer (Modifier-based, no extra lib); Error → inline + retry.
- **Cache strategy:** SQLDelight; serve stale immediately, refresh in background; TTL matches ESI `Cache-Control`. Pull-to-refresh and on-foreground refresh.
- **Navigation:** M3 `NavigationBar` 4 items (Dashboard / Skills / PI / Jobs), adaptive `NavigationRail` for tablets, 5th slot reserved.
- **Android notifications:** `ForegroundService` + `AlarmManager` for exact `finish_date`.
- **iOS notifications:** `UNUserNotificationCenter` schedules at exact `finish_date`; `BGAppRefreshTask` for best-effort sync.
- **Testing (`commonTest`):** `SkillProgressCalculatorTest`, `EsiResponseParserTest`, `PiStatusUseCaseTest`, `OAuthPkceTest`, `SkillQueueViewModelTest`.
- **Out of scope (MVP):** market data, full assets, corporation/alliance, multi-character, mail, fitting tool. Non-breaking additions planned for later.

## Open questions
- [ ] Per-endpoint ESI pages deferred — create in data-layer plan when each call is wired.
- [ ] EVE application client_id and redirect registration status (dev portal) — needs to happen before Auth plan.
- [ ] Minimum Android API level + iOS deployment target — not stated in spec.
- [ ] Exact SQLDelight schema versioning strategy beyond "cache TTL matches ESI headers".
- [ ] Compose MP iOS maturity: any known showstoppers for M3 components we rely on?

## Wiki pages touched
- Entities: [[Character]], [[Skill Queue]], [[Planet]], [[Industry Job]]
- Concepts: [[OAuth2 PKCE]], [[UiState]], [[SecureStorage]], [[Stale-While-Revalidate Cache]]
- Decisions: [[ADR-001 - KMP Compose Multiplatform]], [[ADR-002 - Material 3 Dark Default]], [[ADR-003 - Voyager and Bottom Navigation]], [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]], [[ADR-005 - Math-Based Skill Progress]], [[ADR-006 - Android Foreground Service]], [[ADR-007 - iOS Local Notifications]], [[ADR-008 - OAuth2 PKCE via System Browser]], [[ADR-009 - UiState Sealed Class with Shimmer]]
- Pattern: [[Math-Based Progress Bar]]
- Screens: [[Screen - Dashboard]], [[Screen - Skills]], [[Screen - PI]], [[Screen - Jobs]]
- Platform: [[Platform - Android]], [[Platform - iOS]]
- ESI: [[ESI Scopes MVP]]
