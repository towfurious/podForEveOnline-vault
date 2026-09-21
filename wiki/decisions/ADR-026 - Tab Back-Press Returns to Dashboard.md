---
title: ADR-026 - Tab Back-Press Returns to Dashboard
type: decision
tags: [adr, android, navigation, voyager, ux]
aliases: [ADR-026, Back Button Fix, PlatformBackHandler]
created: 2026-09-21
updated: 2026-09-21
sources: []
status: active
adr-status: Accepted
---

# ADR-026 — Tab Back-Press Returns to Dashboard

## Status
Accepted 2026-09-21.

## Context
[[Guide - App Store Launch Readiness]]'s P2 backlog carried a known bug since 2026-07-16: pressing the system back button from any bottom-nav tab (Skills/PI/Jobs) exits straight to the launcher instead of returning to the Dashboard tab first. Deprioritized as "minor UX polish, not launch-blocking" while still in closed testing with 12 known testers — revisited once Google Play production access was granted, on the reasoning that a first-time real public user is far more likely to hit and notice it than an opted-in tester already used to the app.

**Root cause, confirmed by reading Voyager 1.1.0-beta03's actual sources** (not guessed from the symptom): `TabNavigator` (`cafe.adriel.voyager.navigator.tab`) wraps its content in a `Navigator` with `onBackPressed` explicitly hardcoded to `null`:
```kotlin
// Voyager's TabNavigator.kt
Navigator(screen = tab, onBackPressed = null, ...) { ... }
```
`NavigatorBackHandler`'s internal guard (`if (onBackPressed != null) { BackHandler(...) }`) never fires as a result — no platform back-press handler is ever registered while a tab is showing, by Voyager's own design. Combined with each `Tab.Content()` in `App.kt` delegating straight to a screen (no nested per-tab `Navigator`/back-stack), there was nothing anywhere in the composition intercepting back — it fell straight through to `MainActivity`'s default `ComponentActivity` behavior, which finishes the activity.

## Decision
A single `BackHandler` registered once at the `MainApp()` level, gated on "not already on the home tab":
```kotlin
// App.kt, inside TabNavigator(DashboardTab) { ... }
val tabNavigator = LocalTabNavigator.current
PlatformBackHandler(enabled = tabNavigator.current != DashboardTab) {
    tabNavigator.current = DashboardTab
}
```
`tabNavigator.current = DashboardTab` is the exact same tab-switch mechanism `PodNavBar`'s own click handler already uses — no new navigation primitive introduced.

**`PlatformBackHandler` is a new `expect`/`actual`** (`platform/PlatformBackHandler.kt`), not a direct `androidx.activity.compose.BackHandler` import in `App.kt`, because `App.kt` lives in `commonMain` and `androidx.activity:activity-compose` is only a Gradle dependency under `composeApp`'s `androidMain` source set (not multiplatform) — same shape as the existing `RequestNotificationPermissionEffect` expect/actual this mirrors. Android actual wraps the real `BackHandler`; iOS actual is a no-op — iOS has no hardware/gesture back button that exits an app to the home screen the way Android's system back does, so there's nothing to intercept there.

## Consequences

**Positive**
- Matches the exact behavior users expect from any Android app with bottom-nav tabs: back returns to home first, exits only from home.
- Root cause is now on record precisely (Voyager's own source, not the vault's earlier "likely a TabNavigator characteristic" guess) — useful if this surfaces again after a Voyager version bump.
- Device-verified end-to-end: Skills → back → Dashboard (app still running) → back → launcher (app exits), confirmed via real `adb` back-press + screenshots, not just code review.

**Negative / watch items**
- Checked for conflict with the one modal surface in the app (Dashboard's settings `ModalBottomSheet`) — no real risk exists in practice: that sheet only opens from the Dashboard tab, where this handler's `enabled` is already `false`, so the two can never compete for the same back-press. Worth re-checking if a sheet/dialog is ever added to a non-Dashboard tab.
- If Voyager ever starts allowing `onBackPressed` to be non-null in `TabNavigator` (a future version), this app-level handler and any newly-enabled Voyager-internal one could both react to the same back-press — no such change is expected, but worth a quick check on the next Voyager bump.

## Alternatives considered
- **Per-tab child `Navigator`s** (giving each tab its own back-stack) — rejected as significant, unneeded restructuring; no tab in this app has internal multi-screen navigation today, so there's no real back-stack to manage, only the tab-switch itself.
- **Overriding `MainActivity.onBackPressedDispatcher` directly** (Android-only, outside Compose) — rejected in favor of the `expect`/`actual` Composable shape, consistent with how this codebase already handles every other platform-specific effect (`RequestNotificationPermissionEffect`, `PlatformBackHandler` now).

## References
- [[Guide - App Store Launch Readiness]] — P2 item this closes.
- [[ADR-003 - Voyager and Bottom Navigation]] — the navigation library and structure this builds on.
