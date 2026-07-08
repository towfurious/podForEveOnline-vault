---
title: ADR-010 — Platform Targets
type: decision
tags: [adr, android, ios, platform]
aliases: []
created: 2026-06-30
updated: 2026-06-30
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-010 — Platform Targets

## Status
Accepted 2026-06-30.

## Context
The design spec did not specify a minimum Android API level or iOS deployment target. These must be fixed before creating the Gradle project so that `minSdk`, `targetSdk`, `compileSdk`, and the iOS deployment target in the Xcode project are set correctly from the start.

## Decision
- **Android `minSdk`: 28** (Android 9.0 Pie, 2018). Cleartext traffic off by default, `EncryptedSharedPreferences` stable, `FOREGROUND_SERVICE` permission semantics well-established.
- **Android `targetSdk`**: latest stable at build time (start at 35, bump annually).
- **Android `compileSdk`**: same as `targetSdk`.
- **iOS deployment target: 16.0**. Covers `UNUserNotificationCenter`, `ASWebAuthenticationSession`, and satisfies Compose Multiplatform 1.7+ stable iOS tier requirement.

## Consequences
**Positive**
- API 28 covers ~88% of active Android devices in 2026; cleartext-off default removes a footgun; ForegroundService + EncryptedSharedPreferences story is clean.
- iOS 16.0 covers ~95%+ of iOS devices in 2026; no iOS 17-specific SwiftUI APIs needed.
- Compose MP stable iOS tier starts at 16.0 — we are aligned.

**Negative**
- Drops devices below API 28 (~12% tail) and below iOS 16 (~5% tail).

## Alternatives considered
- **Android API 26** — ~94% coverage but misses cleartext-off defaults and some ForegroundService refinements; coverage gain not worth the regression.
- **Android API 30** — cleaner scoped storage; cuts ~10% of devices without adding anything we use.
- **iOS 17.0** — no Compose MP benefit; cuts ~15% of devices needlessly.

## References
- [[Platform - Android]]
- [[Platform - iOS]]
- [[ADR-006 - Android Foreground Service]]
- [[ADR-007 - iOS Local Notifications]]
