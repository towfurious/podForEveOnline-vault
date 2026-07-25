---
title: ADR-022 - Demo Mode
type: decision
tags: [adr, android, ios, auth, play-store]
aliases: [AuthState.Demo, DemoData]
created: 2026-07-24
updated: 2026-07-24
sources: []
status: active
adr-status: Accepted
---

# ADR-022 — Demo Mode: preview the app with static sample data, no EVE account required

## Status
Accepted 2026-07-24.

## Context
Google Play's "Sign in details" review requirement ([support.google.com/googleplay/android-developer/answer/15748846](https://support.google.com/googleplay/android-developer/answer/15748846)) mandates reusable, always-valid login credentials for any non-Google third-party login before a listing can be reviewed. PodForEve gates 100% of its real content behind EVE Online SSO (OAuth2 PKCE via Chrome Custom Tabs, see [[OAuth2 PKCE]]) — there's no Google-native login exempt from this rule, so the form's honest answer was "Yes, restricted," which normally means handing reviewers real login credentials.

The user asked directly whether that's really required, and to check how other EVE Online third-party apps handle it. Research found: (a) Google's own docs confirm reusable creds are mandatory for any non-"Sign in with Google" flow, and (b) [EVECompanion](https://developers.eveonline.com/docs/community/evecompanion/) — an existing EVE companion app — lists **"Demo Mode: Explore all features without logging in"** as a feature, sidestepping the problem entirely rather than sharing account credentials with a third party.

## Decision
Added `AuthState.Demo` as a fourth branch alongside `Unauthenticated`/`Loading`/`Authenticated`/`Error` (`AuthState.kt`). Entry point: a "Try Demo Mode" `TextButton` on `LoginScreen`, calling `AuthRepository.enterDemoMode()`. Exit: either a persistent top banner (`DemoModeBanner` — "Demo Mode — sample data" / "Exit") shown above all four tabs, or a relabeled "Exit Demo Mode" row in the Dashboard settings sheet (replaces "Log out" while `isDemo` is true) — both call `AuthRepository.exitDemo()`.

All 4 ViewModels (`DashboardViewModel`, `SkillQueueViewModel`, `PlanetViewModel`, `IndustryJobViewModel`) branch on `authRepository.authState.value` before touching their repository: in `Demo`, they emit `UiState.Success(DemoData.x)` directly from a new `composeApp/.../demo/DemoData.kt` object, never calling ESI or SQLDelight. `enterDemoMode()`/`exitDemo()` are plain `MutableStateFlow` sets — no token exchange, no DB write — so exiting is a no-op cache-wise, unlike real `logout()` (which wipes 4 SQLDelight tables, see [[OAuth2 PKCE]] "Logout").

Net effect on the Play Console form: since Demo Mode exposes every screen fully without any login at all, a reviewer can evaluate the entire app via "Try Demo Mode" — no EVE Online account, throwaway or otherwise, ever needs to be shared with Google.

## Consequences

**Positive**
- No real (or even disposable) EVE Online account credentials are ever handed to a third party for store review — closes the concern the user raised directly.
- Doubles as a real product feature, not just a review workaround: any user unsure about granting EVE SSO login can preview the app first.
- Zero ESI/DB writes in Demo Mode — can't corrupt real cached data, can't trip the ESI rate-limit budget ([[ADR-018 - ESI HTTP Essentials]]), can't schedule spurious notifications.
- Same mechanism will cover Apple App Store Connect's equivalent "demo account" review-notes field once iOS work resumes — no separate solution needed per platform.

**Negative / watch items**
- `DemoData` is a Kotlin `object`; its `now` field is computed once at first access and never refreshed, so relative-time text ("2h ago") will drift stale the longer the process lives without being killed. Accepted — Demo Mode is a short review/preview session, not a long-lived one.
- Adds a small amount of duplicated `when (authState) { is Demo -> …; is Authenticated -> …; else -> … }` branching across 4 ViewModels. Accepted — matches the pre-existing pattern where each ViewModel already independently unwrapped `AuthState.Authenticated` the same way; containing demo-awareness at the ViewModel layer (rather than pushing a sentinel `characterId` into the repositories) keeps `CharacterRepository`/`SkillQueueRepository`/`PlanetRepository`/`IndustryJobRepository` and `AppDatabase` completely untouched by this feature.
- `DashboardSuccess` had a pre-existing, baselined `LongMethod` detekt finding (`composeApp/detekt-baseline.xml`); adding the `isDemo` parameter changed its signature text and invalidated that baseline entry. Fixed properly rather than re-suppressed: extracted the settings-bottom-sheet UI into its own `DashboardSettingsSheet` composable (reduced 178→128 lines); the remaining ~128 lines are unrelated pre-existing dashboard-content bulk, re-baselined as-is via `detektBaseline`.
- `Planet.status()` is dead code whenever `colony != null` (`PiScreen.kt` only renders it when `colony == null`) — caught during selfcheck before shipping. `DemoData.planets`' three entries are tuned instead to hit `ExtractorCountdown`'s three actually-rendered color states (stopped/amber/healthy).

## Alternatives considered
- **Dedicated throwaway EVE Online Alpha account** (free, real, given directly to Google) — rejected per the user's own instinct that handing any real game-account credentials to a third party is unnecessary risk when a cleaner path exists; EVECompanion's precedent confirmed one does.
- **Mock network layer** (WireMock/OkHttp interceptor faking the ESI backend) — rejected as far more invasive than needed; Demo Mode's only real job is populating the same `UiState<T>` shapes the screens already render, which hardcoded domain-model data does directly.
- **Sentinel `characterId` (e.g. `-1L`) routed through the real repositories/DB** — rejected: would mean real SQLDelight reads/writes and still-attempted real ESI calls unless every repository method special-cased the sentinel, spreading demo-awareness into the data layer instead of containing it at the ViewModel boundary.

## References
- [[Guide - App Store Launch Readiness]] — P0 item on the Play Console sign-in-details requirement.
- [[OAuth2 PKCE]] — the `AuthState` state machine this extends.
- [[ADR-018 - ESI HTTP Essentials]] — the rate-limit budget Demo Mode never touches.
- [Play Console "Sign in details" requirements](https://support.google.com/googleplay/android-developer/answer/15748846)
- [EVECompanion](https://developers.eveonline.com/docs/community/evecompanion/) — the precedent that motivated this approach.
