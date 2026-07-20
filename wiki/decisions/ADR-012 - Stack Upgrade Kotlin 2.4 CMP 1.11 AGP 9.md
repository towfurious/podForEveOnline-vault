---
title: ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9
type: decision
tags: [adr, kmp, kotlin, compose-mp, android, agp]
aliases: []
created: 2026-07-09
updated: 2026-07-17
sources: []
status: active
---

# ADR-012 — Stack Upgrade: Kotlin 2.4.0 + CMP 1.11.1 + AGP 9.2.0

## Status
Accepted 2026-07-09

> **Addendum (2026-07-17)**: the deferred migration below landed — see [[ADR-016 - AGP KMP Library Plugin Migration]]. The `libraryVariants`/Voyager risk flagged in this ADR's Consequences was tested first, empirically, before anything else; it did not reproduce under `com.android.kotlin.multiplatform.library` + `android.newDsl=true`. `android.builtInKotlin`/`android.newDsl` are `true` again as of that migration.

## Context

Previous stack (Kotlin 2.2.21 + CMP 1.9.3 + AGP 8.13.2 + Gradle 8.14.5) was current as of 2026-07-09 but was two major releases behind in Kotlin and CMP. The user requested the maximum compatible latest stack.

Key compatibility constraint: CMP 1.11.1 requires Kotlin 2.4.0; AGP 9.0 changed the plugin model for KMP modules.

## Decision

Upgrade to:

| Component | Version |
|---|---|
| Kotlin | 2.4.0 |
| Compose Multiplatform | 1.11.1 |
| AGP | 9.2.0 |
| Gradle | 9.4.1 |
| Ktor | 3.5.1 |
| Koin | 4.1.1 |
| kotlinx-datetime | 0.8.0 |
| kotlinx-serialization | 1.11.0 |
| kotlinx-coroutines | 1.11.0 |
| haze | 1.7.2 |
| coil3 | 3.5.0 |
| SQLDelight | 2.3.2 |

**AGP 9.0 + KMP compat decision**: AGP 9.0 deprecated using `com.android.library` alongside `org.jetbrains.kotlin.multiplatform`. The correct fix is to migrate to `com.android.kotlin.multiplatform.library`. We chose to defer this migration and keep the old plugin pair, using `android.builtInKotlin=false` + `android.newDsl=false` in `gradle.properties`. Rationale: migration requires a DSL refactor across two build files; the deprecation becomes an error in AGP 10.0 (not 9.x); JetBrains has not published a final migration guide as of 2026-07-09.

## Consequences

**Positive**
- Kotlin K2 compiler (2.4.0) — faster incremental builds, improved error messages.
- CMP 1.11.1 — latest stable iOS/Android parity.
- Ktor 3.x — updated API (`.body()` pattern already used, no source changes needed).
- Koin 4.x — minor API improvements, still compatible with Voyager `voyager-koin` artifact.
- `iosX64` (Intel simulator) target removed — CMP 1.11.1 dropped those artifacts; only Apple Silicon (`iosArm64`, `iosSimulatorArm64`) supported.

**Negative / watch items**
- `android.builtInKotlin=false` and `android.newDsl=false` are themselves deprecated (will be removed in AGP 10.0). The `newDsl=false` bypass causes Voyager's internal use of `libraryVariants` to work; without it, those calls fail.
- `@Preview` annotation in `commonMain` still imports `org.jetbrains.compose.ui.tooling.preview.Preview` (deprecated in CMP 1.11 in favour of `androidx.compose.ui.tooling.preview.Preview` from `org.jetbrains.compose.ui:ui-tooling-preview`). Workaround: compile-only warning, does not affect runtime.

## Alternatives considered

**Stay on AGP 8.x**: would have avoided the KMP plugin compat issue but foregone R8/D8 and build speed improvements in AGP 9.x.

**Migrate to `com.android.kotlin.multiplatform.library` now**: the correct long-term path, but requires replacing the `android {}` DSL block with the new `androidLibrary {}` block in both `shared` and `composeApp` modules — a non-trivial refactor with risk of surfacing new issues.

## References

- https://kotl.in/gradle/agp-new-kmp
- CMP 1.11 release notes (removed `iosX64`; deprecated `compose.components.uiToolingPreview`)
- [[Guide - Compose Multiplatform Maturity Findings]] — this ADR's version-lockstep chain and the pre-flagged Voyager/`libraryVariants` risk are cited there as maturity findings.
