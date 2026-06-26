---
title: ADR-001 — KMP Compose Multiplatform
type: decision
tags: [adr, architecture, kmp, compose-mp]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-001 — KMP + Compose Multiplatform

## Status
Accepted 2026-04-24.

## Context
The app targets Android and iOS. The UI surface is small (4 screens) and the business logic — ESI access, OAuth, caching, progress math — is platform-agnostic. The user wants to maintain one codebase without dropping UI quality.

## Decision
Use **Kotlin Multiplatform** for the shared domain + data layer, and **Compose Multiplatform** for the shared UI layer. Module split:

```
/composeApp    — shared UI (screens + ViewModels), Compose MP
/shared        — pure KMP: domain + data (Ktor, SQLDelight, Koin)
/androidApp    — Android entry point + ForegroundService
/iosApp        — iOS entry point + AppDelegate
```

## Consequences
**Positive**
- Single codebase for 4 screens, ViewModels, use cases, repositories.
- Kotlin-only stack — no JS/TS, no Dart.
- Platform-specific code isolated to `expect`/`actual` (e.g. [[SecureStorage]], notifications).

**Negative**
- Compose MP on iOS is production-ready but less battle-tested than native UIKit/SwiftUI — subtle rendering bugs or performance gaps possible.
- Debugging stack traces across the Kotlin/Native boundary is uglier than JVM-only.
- Team must be comfortable in both Android and iOS toolchains (Xcode for iOS app shell).

## Alternatives considered
- **Fully native (Kotlin for Android, Swift for iOS).** Rejected: doubles the work and drifts out of sync; the 4-screen MVP doesn't justify it.
- **Flutter.** Rejected: Dart instead of Kotlin; weaker ESI-domain code reuse if we ever share with a Kotlin backend.
- **React Native.** Rejected: JS bridge ceremony; worse for our compute-light, IO-heavy workload than Kotlin+Ktor.
- **KMP with native UI (shared logic only).** Rejected: spec explicitly wants shared UI; Compose MP removes the parallel-UI-implementation tax.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
