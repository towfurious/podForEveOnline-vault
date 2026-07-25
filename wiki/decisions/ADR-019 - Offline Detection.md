---
title: ADR-019 - Offline Detection
type: decision
tags: [adr, kmp, expect-actual, android, ios, layer-platform]
aliases: [ConnectivityChecker, OfflineBanner]
created: 2026-07-21
updated: 2026-07-24
sources: []
status: active
adr-status: Accepted
---

# ADR-019 — Offline detection: polling ConnectivityChecker, not push-based callbacks

## Status
Accepted 2026-07-21.

## Context
[[Guide - App Store Launch Readiness]] P1 #6: the app is fully ESI-dependent but had no connectivity awareness at all — a dropped connection surfaced only as a generic [[UiState]]`.Error` after a failed request, with no proactive signal. The guide's own technical note suggested "Android `ConnectivityManager`, iOS `NWPathMonitor`" — both are native *push*-based observers (callback registration on a change event).

## Decision
`expect class ConnectivityChecker { fun isOnline(): Boolean }` (`shared/.../platform/ConnectivityChecker.kt`) — a **synchronous, single-shot** check, not a registered observer:
- **Android** (`ConnectivityChecker.android.kt`): `ConnectivityManager.activeNetwork` → `getNetworkCapabilities()` → `NET_CAPABILITY_INTERNET && NET_CAPABILITY_VALIDATED`. No callback registration.
- **iOS** (`ConnectivityChecker.ios.kt`): `SCNetworkReachabilityCreateWithName(null, "esi.evetech.net")` + `SCNetworkReachabilityGetFlags` (`SystemConfiguration` framework), checked for `kSCNetworkReachabilityFlagsReachable` without `kSCNetworkReachabilityFlagsConnectionRequired`. Deliberately **not** `Network.framework`'s `NWPathMonitor` — that's a Swift-only wrapper with no Objective-C header, so Kotlin/Native's cinterop (which parses ObjC/C headers, not Swift modules) can't see it directly; the underlying C API (`nw_path_monitor_create` + friends) is reachable but needs block/callback interop across the K/N boundary for an update handler. `SCNetworkReachability`'s synchronous flag-check needs none of that.

On top, `ConnectivityObserver` (`shared/.../platform/ConnectivityObserver.kt`) wraps the synchronous checker in a **5-second poll loop** (`flow { while (true) { emit(checker.isOnline()); delay(5_000) } }.distinctUntilChanged()`) — the same class works identically on both platforms, rather than writing two different callback-registration lifecycles (`NetworkCallback` register/unregister on Android, a dispatch-queue-driven update handler on iOS) for a banner that only needs to react within a few seconds, not instantly.

Wired at the very top of `App()` (`composeApp/.../App.kt`), above the splash/login/main-app routing — so [[OAuth2 PKCE|login]] (also network-dependent) gets the same banner as the four data screens. `OfflineBanner` (`composeApp/.../ui/component/OfflineBanner.kt`) uses `MaterialTheme.colorScheme.errorContainer`/`onErrorContainer` so it renders correctly across all 5 [[ADR-013 - Faction Color Themes|faction themes]] with zero new theme tokens, matching that ADR's own established pattern.

## Consequences

**Positive**
- One shared implementation (`ConnectivityObserver`) instead of two divergent native-callback lifecycles to write and maintain.
- Synchronous `isOnline()` is trivial to reason about and needs no lifecycle management (no register/unregister to get wrong, no leaked callback).
- Global placement means every screen — including the not-yet-authenticated [[Screen - Dashboard|login]] state — benefits without per-screen wiring.

**Negative / watch items**
- Up to ~5 s detection latency by design (documented as an explicit tradeoff, not an oversight) — acceptable for a banner, not a fit for anything needing near-instant reaction.
- iOS's reachability target is hardcoded to `esi.evetech.net` specifically (not a generic "any internet" check) — correct for this app (the only host that matters is ESI) but worth remembering if a future feature depends on a different host's reachability.
- Neither platform's check distinguishes "connected to Wi-Fi with no internet" perfectly in all edge cases (captive portals, etc.) — Android's `NET_CAPABILITY_VALIDATED` covers the common case; iOS's reachability flags are a coarser signal. Acceptable for a "you're probably offline" banner, not treated as authoritative elsewhere in the app (ESI calls still get their own real error handling regardless of what this banner shows).

**Real bug found on-device (2026-07-24), by eye, not by any automated check**: user spotted from a screenshot that `OfflineBanner`'s red background and text drew *behind* the status bar (into the clock/icon row's own space) instead of starting below it — every other screen in this app already applies `WindowInsets.statusBars` via its own `Scaffold(contentWindowInsets = ...)`, but the banner sits *outside* any screen's `Scaffold`, at the very top of `App()` itself, so it never got that treatment. Fixed with the same `Modifier.windowInsetsPadding(WindowInsets.statusBars)` on the banner's `Surface`. Verified with a real airplane-mode toggle on a Pixel 10 Pro XL both directions (offline → banner appears correctly below the status bar; back online → banner disappears cleanly) — not just a static layout read.

**Real bug found on-device (2026-07-22), not by any static check run before this ADR was first accepted**: `ConnectivityChecker.android.kt`'s `ConnectivityManager.getActiveNetwork()` call requires `android.permission.ACCESS_NETWORK_STATE`, which was never added to `AndroidManifest.xml` — fatal `SecurityException` on launch, every single time, on a real device. Root cause of it slipping through: the original verification pass ran `shared`/`composeApp`'s `ktlintCheck`/`detekt` and both unit-test suites, but never `androidApp:lintDebug` — the one task in this project that runs real Android Lint (`shared`/`composeApp` lost their own Lint-in-`check` integration under [[ADR-016 - AGP KMP Library Plugin Migration]]'s new plugin, `androidApp` is unaffected and still has it). Android Lint's `MissingPermission` check would have caught this immediately, the same way it already caught real findings in this codebase before (see `log.md` 2026-06`/`2026-07 lint sweeps). Fixed: added `<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />` to `AndroidManifest.xml` (a normal-protection-level permission — granted automatically at install, no runtime prompt needed, unlike `POST_NOTIFICATIONS`). Confirmed via `androidApp:assembleDebug androidApp:lintDebug` clean afterward.

## Alternatives considered
- **Push-based `ConnectivityManager.NetworkCallback` (Android) + `NWPathMonitor`/`nw_path_monitor_*` (iOS)**: rejected for this pass — instant reaction isn't needed for a banner, and it would mean maintaining two structurally different callback lifecycles (register/unregister semantics differ enough between the two APIs) instead of one shared polling class. Worth revisiting only if a future feature needs sub-second connectivity reaction.
- **`Network.framework`'s Swift `NWPathMonitor` directly**: not viable at all from Kotlin/Native — no ObjC header for cinterop to see.
- **Skip iOS-specific reachability entirely, rely only on request-failure detection** (already partially true via [[EsiErrorMapper|util.EsiErrorMapper]]'s `isNetworkError`): rejected — the guide explicitly asked for proactive detection, and a request-failure-only signal can't show a banner *before* the user taps into a screen and waits for a timeout.

## References
- [[Guide - App Store Launch Readiness]] — P1 #6.
- [[ADR-018 - ESI HTTP Essentials]] — the paired P1 item done in the same pass; that ADR's `EsiErrorMapper` network-error path remains the fallback when this banner's coarse signal is wrong.
- [[ADR-013 - Faction Color Themes]] — theme tokens `OfflineBanner` reuses.
