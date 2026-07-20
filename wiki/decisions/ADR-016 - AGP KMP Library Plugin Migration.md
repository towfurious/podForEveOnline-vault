---
title: ADR-016 - AGP KMP Library Plugin Migration
type: decision
tags: [adr, kmp, kotlin, android, agp, expect-actual]
aliases: []
created: 2026-07-17
updated: 2026-07-17
sources: []
status: active
---

# ADR-016 — Migrate shared + composeApp to com.android.kotlin.multiplatform.library

## Status
Accepted 2026-07-17.

## Context
[[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]] deferred migrating off `com.android.library` + `org.jetbrains.kotlin.multiplatform` (deprecated together since AGP 9.0) via two `gradle.properties` bypass flags, `android.builtInKotlin=false` and `android.newDsl=false`. Those flags lose their effect in AGP 10.0. The user asked to do the migration now rather than let it accumulate against a growing codebase.

Two real risks surfaced during investigation, both resolved:
1. **`com.android.kotlin.multiplatform.library` does not support `BuildConfig` generation at all** (it's variant-agnostic; `BuildConfig` needs build types/flavors) — breaking [[ADR-011 - Secrets via expect-actual and local.properties]]'s exact mechanism for injecting the ESI OAuth `client_id`.
2. ADR-012 itself documented that `android.newDsl=false` was needed for **Voyager's internal `libraryVariants` usage** to work. This was treated as the highest-risk unknown and tested first, empirically, before any other verification — see Verification below.

## Decision
- **`shared` and `composeApp`**: `com.android.library` → `com.android.kotlin.multiplatform.library`. The new plugin has no top-level `android {}` block — all config (`namespace`, `compileSdk`, `minSdk`, `androidResources`, `withHostTestBuilder`, target-specific `compilerOptions`) moves inside `kotlin { android { ... } }`. No `buildFeatures {}` block exists in this plugin's DSL at all (confirmed via two independent official-doc fetches, kotlinlang.org and developer.android.com) — Compose Multiplatform's own plugins drive Compose independently of AGP's `buildFeatures.compose` flag, so nothing needed to replace it.
- **`androidApp`**: stays a plain `com.android.application` (untouched build types/proguard/Chucker debug-release split) — only drops the now-redundant `org.jetbrains.kotlin.android` plugin, relying on AGP 9's built-in Kotlin support instead.
- **`gradle.properties`**: `android.builtInKotlin` / `android.newDsl` flipped to `true`. This is atomic across all three modules — Gradle configures every subproject before running any task, so a partial migration is a hard configuration-time error project-wide, not a per-module rollout.
- **ESI_CLIENT_ID replacement**: a small custom Gradle task (`GenerateEsiConfigTask` in `shared/build.gradle.kts`) generates `com.podforeve.tracker.auth.GeneratedEsiConfig.CLIENT_ID`, reading `local.properties`'s `esi.client_id` exactly like the old `buildConfigField` did (same graceful `""` default when absent). Chosen over the community `com.codingfeline.buildkonfig` plugin: this project already hand-rolls the same `Properties()`/`local.properties`-reading idiom in [[ADR-011 - Secrets via expect-actual and local.properties]] rather than reaching for a plugin, a custom task needs zero new dependencies, and its output directory (`build/generated/esiConfig/...`) already matches the existing ktlint exclude filter with no config changes — BuildKonfig's default output path would not have.
- `composeApp/build.gradle.kts`'s `debugImplementation(libs.compose.ui.tooling)` became `"androidRuntimeClasspath"(libs.compose.ui.tooling)` — the new plugin has no build variants, so variant-scoped configurations like `debugImplementation` don't exist; tooling-only deps go straight on the runtime classpath. `androidApp`'s own `debugImplementation(libs.compose.ui.tooling)` is untouched (real variants still exist there).
- **`gradle/libs.versions.toml`**: added `android-kotlin-multiplatform-library`, reusing the existing `agp` version ref (the plugin's versions track AGP's release train; 9.2.0 exceeds its 8.10.0 minimum). Left the now-unused `android-library`/`kotlin-android` catalog entries in place — that cleanup is unrelated to this change.
- **CI** (`.github/workflows/ci.yml`): `shared:lintDebug composeApp:lintDebug shared:testDebugUnitTest` → `shared:check composeApp:check`. The old task names don't exist under the variant-agnostic plugin; `check` is Gradle's stable lifecycle task regardless of how a plugin names things underneath, so this doesn't need re-guessing on a future plugin update. `androidApp:lintDebug` is untouched.

## Consequences

**Positive**
- Removes deprecated-and-expiring `gradle.properties` bypass flags before AGP 10.0 forces the issue.
- Secret injection (`GeneratedEsiConfig`) is simpler than `BuildConfig` ever was — no `buildFeatures.buildConfig = true` ceremony, just a task and a source-set wire-up.
- `androidApp` no longer carries a redundant standalone Kotlin plugin alongside AGP's built-in support.

**Negative / watch items**
- **Android Lint coverage on `shared`/`composeApp` changed**: the new plugin exposes `lintAnalyzeAndroidHostTest` but no equivalent to the old `lintDebug` wired into `check` — confirmed via `:shared:tasks --all` and a `:shared:check --dry-run` trace (only ktlint, detekt, and tests showed up, no lint task). `androidApp`'s Android Lint is completely unaffected (still a plain `com.android.application`). This is a real, if secondary, coverage gap on the two migrated modules — worth revisiting if this plugin's lint story matures, or if `androidApp`'s lint start missing something these two modules used to independently catch (recall [[Guide - App Store Launch Readiness]]'s Android Lint `MissingPermission` findings were specifically in `shared`'s androidMain code).
- **SQLDelight's `verifyMigrations = true` had never actually been exercised**: adopting `shared:check` (instead of hand-picked task names) surfaced that `verifyCommonMainAppDatabaseMigration` had no baseline schema file to check against — a pre-existing gap unrelated to this migration, just never triggered before since the project's prior verification commands never invoked `check`/this specific task. Fixed by running `:shared:generateCommonMainAppDatabaseSchema` once to produce the missing `1.db` baseline (a safe, additive, one-time action — not a behavior change).
- Reduces this project's own migration surface for any *future* AGP major bump, since the deprecated bypass path is gone — but the new plugin is meaningfully younger/less-documented than classic `com.android.library`; expect rougher edges (see the compile errors hit during implementation, both from following the fetched official doc's exact-but-inapplicable syntax: `compilerOptions.configure { }` doesn't resolve inside `kotlin { android { } }` — plain `compilerOptions { }` does; `androidMain.kotlin.srcDir(...)` as a standalone statement doesn't resolve — must be inside an `androidMain { }` block).

## Alternatives considered
- **Defer further** (status quo per ADR-012): rejected per explicit user request to do this before the codebase grows more.
- **BuildKonfig plugin** for the secret-injection replacement: rejected, see Decision above.
- **Migrate `androidApp` alone first, leave `shared`/`composeApp` on the old plugin**: not viable — `android.newDsl`/`android.builtInKotlin` are project-global; flipping them without migrating `shared`/`composeApp` off `com.android.library` first is a hard configuration-time error for the whole build, not a partial-success state.

## Verification
Sequenced deliberately with the highest-risk unknown first, not left to the end:
1. Migrated all files together (per the atomicity constraint above), then `:shared:tasks --all` / `:composeApp:tasks --all` to find real post-migration task names (`testAndroidHostTest`, `lintAnalyzeAndroidHostTest`, `check`, etc. — none of this was documented anywhere findable).
2. Got to a compiling `androidApp:assembleDebug` first and **device-verified Voyager specifically** on a Pixel 10 Pro XL before any further work: all 4 bottom-nav tabs (Dashboard/Skills/PI/Jobs) and the theme-switcher `ModalBottomSheet`, plus confirming the existing login session and live ESI data fetch (ISK/SP/skill queue/PI/jobs) all rendered correctly. ADR-012's documented `libraryVariants` risk **did not reproduce** — navigation and the bottom sheet both worked cleanly.
3. Only after that: full suite (`shared:check composeApp:check androidApp:lintDebug shared:iosSimulatorArm64Test shared:compileKotlinIosSimulatorArm64 androidApp:assembleDebug`), confirmed clean via actual test-result XML counts (not just "BUILD SUCCESSFUL") — zero failures/errors across both the Android host-test suite and the iOS Kotlin/Native suite.
4. Verified the exact new CI command sequence locally before considering the workflow update done.

## References
- [[ADR-011 - Secrets via expect-actual and local.properties]] — the secret-injection convention this migration had to preserve.
- [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]] — where this migration was originally deferred, and where the Voyager risk was first documented.
- [[Guide - App Store Launch Readiness]] — P1 #7 (CI) references this migration's task-naming changes.
- [[Guide - Compose Multiplatform Maturity Findings]] — this migration's rough edges (undocumented task names, non-compiling official-doc sample, the pre-flagged Voyager risk) are cited there as a build-tooling maturity finding.
- https://developer.android.com/kotlin/multiplatform/plugin
- https://kotlinlang.org/docs/multiplatform/multiplatform-project-agp-9-migration.html
