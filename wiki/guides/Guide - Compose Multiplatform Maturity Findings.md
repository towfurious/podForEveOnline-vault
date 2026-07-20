---
title: Guide - Compose Multiplatform Maturity Findings
type: guide
tags: [guide, kmp, compose-mp, agp, android, ios]
aliases: [CMP Maturity Findings, KMP Maturity Findings]
created: 2026-07-20
updated: 2026-07-20
sources: []
status: active
---

# Guide — Compose Multiplatform Maturity Findings

## Goal
PodForEve's actual purpose is to find out what Compose Multiplatform is like *right now* — a hands-on evaluation of the framework's current maturity, not only a normal production app built under normal shipping-risk constraints. Hitting and understanding friction is largely the point, not an obstacle to route around. This page pulls that evidence together in one place instead of leaving it scattered across individual ADRs and `log.md` entries. **Living document** — add to it whenever a session surfaces a new CMP/KMP-specific finding, good or bad.

## How to read this
Five categories of finding, roughly in the order they've bitten this project. Each bullet is a synthesis with a link to the ADR or log entry that has the full detail — this page doesn't repeat that detail, it connects it.

## Version lockstep and churn
Kotlin, Compose Multiplatform, AGP, Ktor, and Koin don't version independently in practice — bumping one for a real reason (a bug fix, a feature, matching a third-party library's ABI) regularly forces bumping several others in the same pass, each with its own real breaking changes.
- Kotlin went `2.0.21 → 2.1.21 → 2.2.21 → 2.4.0` and CMP `1.6.11 → 1.7.3 → 1.9.3 → 1.11.1` across four separate rounds this project's lifetime, each pairing forced by the other (e.g. CMP 1.7.3 was "the paired release for Kotlin 2.1.x"; Kotlin 2.1.21 was needed for Xcode 16.3+ support). See the `log.md` entries for 2026-07-08 and 2026-07-09 ("Kotlin 2.1.21 + CMP 1.7.3", "Kotlin 2.2.21 + CMP 1.9.3 + haze blur", "Full stack upgrade").
- One of those rounds forced a real breaking-change migration unrelated to the feature being worked on: `kotlinx.datetime.Clock`/`Instant` moved to `kotlin.time` in kotlinx-datetime 0.7.1, requiring import changes across 10 files just to keep compiling.
- Third-party libraries can silently desync from this treadmill: Chucker 4.3.1 was compiled against newer Kotlin metadata than the project's then-current Kotlin Gradle Plugin understood, forcing a downgrade to 4.1.0 — only fixed for real once Kotlin reached 2.4.0 months later (`log.md` 2026-07-08 and 2026-07-17).

## Build tooling: the AGP 9 plugin transition
AGP 9.0 deprecated running `com.android.library` in the same module as the Kotlin Multiplatform plugin, replacing it with a dedicated `com.android.kotlin.multiplatform.library` plugin — see [[ADR-016 - AGP KMP Library Plugin Migration]] for the full migration.
- The new plugin is a real regression in a few concrete ways, not just a rename: **no `BuildConfig` support at all** (this project replaced it with a hand-rolled Gradle task), no top-level `android {}` block, and — confirmed still true post-migration — **Android Lint isn't wired into its `check` lifecycle task**, a real coverage gap accepted as a known limitation rather than solved.
- The official JetBrains migration documentation's own sample syntax didn't compile as written (`compilerOptions.configure { }` inside `kotlin { android { } }` doesn't resolve) — the working syntax had to be found empirically, not by following the docs.
- This project's own ADR-012 had flagged a specific real regression risk from this exact change (Voyager's internal use of `libraryVariants` breaking under the new DSL) months before attempting the migration — a genuine first-hand example of the ecosystem's rough edges being visible even before you hit them yourself. It didn't reproduce when finally tested, but it was a real enough risk to gate the whole migration behind a device-verification step before doing anything else.

## Platform-parity ceilings
Some native-platform capabilities are architecturally unreachable from Compose Multiplatform, not just missing a library version.
- iOS 26's "Liquid Glass" design language is rendered by the system through native `TabView`/`NavigationStack`/toolbar — CMP's Skia-based rendering never touches that chrome, so no dependency bump unlocks it. The only real path is a hybrid architecture handing navigation chrome to a native SwiftUI shell while Compose renders screen content only — a genuine architecture migration, not a version bump. Full detail and the third-party visual-approximation fallback options: [[Guide - App Store Launch Readiness]]'s P2 section.

## Third-party ecosystem lag
Beyond CMP/KMP itself, the surrounding library ecosystem shows real growing pains.
- Multiple dependencies sat on alpha/beta releases with no stable line for extended periods (Voyager, `androidx-security-crypto` at one point) — and the *information* about their state was itself unreliable: WebSearch summaries gave contradictory or wrong "latest version" claims for Koin, detekt, kotlinx-coroutines, kotlinx-serialization, SQLDelight, and Voyager in a single research pass, only resolved by querying Google Maven / Maven Central metadata endpoints directly. See `log.md` 2026-07-17 ("Dependency audit") and the 2026-07-17 `meta` correction entry.
- `PullToRefreshBox` wasn't available in CMP 1.6.11's commonMain at all — had to be worked around with a plain `Box`, later adopted once CMP matured past that version (`log.md` 2026-06-30, "First Android build").

## What's solid
Findings shouldn't skew one direction — plenty of this project's stack has been genuinely reliable.
- The shared business-logic layer (Ktor, SQLDelight, Koin, coroutines, the full OAuth2 PKCE + EVE SSO flow) has not produced a single cross-platform-specific bug across the project's life — issues traced to real code, not the multiplatform abstraction leaking.
- Once a version-lockstep round *is* completed and verified, it stays solid — no regressions have resurfaced from a completed migration; the pain is real but finite per round, not ongoing background noise.
- Fully custom UI — the hand-traced splash screen animation, the neon-glow card treatment ([[ADR-017 - Neon Outline Card and Icon Treatment]]), Haze-based frosted glass, spring-physics nav pill — all render correctly and identically-authored on both Android and iOS. This is CMP's actual value proposition working as advertised when it works.
- The Android target specifically is, underneath, real Android development on the same runtime/tooling as a native app — the multiplatform-specific risk concentrates on the *shared-UI* layer (Compose Multiplatform itself) and the iOS target, not on Android's own maturity.

## Overall assessment
Production-viable today for an app like this one — a utility/companion app that doesn't need bleeding-edge native OS integration — but not "boring" the way native-only Android or native-only iOS development is. The core shared-logic value proposition (write business logic once) is comparatively mature; nearly all of the friction found so far concentrates specifically in the newer *shared-UI* layer (Compose Multiplatform itself) and in the AGP build-tooling seam around it, not in KMP's original, more conservative "shared logic, native UI" use case. Expect to pay a real, recurring tooling-churn tax on every stack upgrade, and expect a hard ceiling the moment an app wants deep native OS-level polish (Liquid Glass being the concrete example found here) rather than a Material-you-can-theme cross-platform look.

## Related
- [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]]
- [[ADR-016 - AGP KMP Library Plugin Migration]]
- [[ADR-017 - Neon Outline Card and Icon Treatment]]
- [[Guide - App Store Launch Readiness]]
- [[ADR-001 - KMP Compose Multiplatform]]
