# Index

> Catalog of every wiki page. Maintained by the LLM agent per `[[CLAUDE]]` §5.
> One line per page: `- [[Page Title]] — one-line description`.

## Entities
- [[Character]] — the single EVE pilot the app tracks.
- [[Skill Queue]] — ordered list of training skills; source of Skills screen + Dashboard training widget.
- [[Planet]] — a PI colony; status is derived as Active / Needs Attention / Idle.
- [[Industry Job]] — active research or manufacturing job.

## Concepts
- [[OAuth2 PKCE]] — public-client OAuth2 extension we use for EVE SSO.
- [[UiState]] — shared sealed class driving Loading / Success / Error on every screen.
- [[SecureStorage]] — `expect/actual` wrapper over Keystore / Keychain for the refresh token.
- [[Stale-While-Revalidate Cache]] — cache discipline: serve stale instantly, refresh in background.
- [[SQLDelight Migrations]] — versioning strategy: version 1 = MVP schema, `.sqm` files, drop-and-repopulate policy.

## Decisions (ADRs)
- [[ADR-001 - KMP Compose Multiplatform]] — core stack choice.
- [[ADR-002 - Material 3 Dark Default]] — theme and zero custom design-system overhead.
- [[ADR-003 - Voyager and Bottom Navigation]] — navigation lib and 4-item NavigationBar with 5th slot reserved.
- [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]] — shared data/infra libraries.
- [[ADR-005 - Math-Based Skill Progress]] — live progress computed on-device, no polling.
- [[ADR-006 - Android Foreground Service]] — persistent shade countdown + AlarmManager.
- [[ADR-007 - iOS Local Notifications]] — UNUserNotificationCenter + BGAppRefreshTask; no FG-service analog.
- [[ADR-008 - OAuth2 PKCE via System Browser]] — Chrome Custom Tabs / ASWebAuthenticationSession.
- [[ADR-009 - UiState Sealed Class with Shimmer]] — UI state contract and Modifier-based shimmer.
- [[ADR-010 - Platform Targets]] — Android minSdk 28, iOS deployment target 16.0.
- [[ADR-011 - Secrets via expect-actual and local.properties]] — CLIENT_ID injected via BuildConfig (Android) and NSBundle/xcconfig (iOS); never committed.
- [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]] — Kotlin 2.4.0 + CMP 1.11.1 + AGP 9.2.0 + Gradle 9.4.1 + Ktor 3.x; AGP 9.0 KMP compat via bypass properties; migration to `com.android.kotlin.multiplatform.library` deferred.

## Patterns
- [[Math-Based Progress Bar]] — compute progress from start/end timestamps, tick via coroutine, never poll.

## ESI endpoints
- [[ESI Scopes MVP]] — the 5 OAuth scopes for MVP (per-endpoint pages added when wired in data-layer plan).
- [[ESI - Skills - Get Skill Queue]] — GET /v2/characters/{id}/skillqueue/ — requires esi-skills.read_skillqueue.v1.
- [[ESI - Universe - Get Type]] — GET /v3/universe/types/{type_id}/ — public; resolves skill_id → name.

## Screens
- [[Screen - Dashboard]] — portrait, ISK, current training, PI/jobs summary.
- [[Screen - Skills]] — big live progress bar + full queue list.
- [[Screen - PI]] — per-planet cards with derived status chip.
- [[Screen - Jobs]] — list of active research / manufacturing jobs with progress.

## Platform
- [[Platform - Android]] — ForegroundService, AlarmManager, Custom Tabs, Keystore.
- [[Platform - iOS]] — UNUserNotificationCenter, BGAppRefreshTask, ASWebAuthenticationSession, Keychain.

## Guides
- [[Guide - Foundation Setup]] — KMP project scaffold: modules, Gradle config, SQLDelight schema v1, iOS entry stubs.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]] — design spec, approved 2026-04-24.

## Open threads
- Per-endpoint ESI pages (list grows during data-layer plan; two added so far).
- [[Screen - Dashboard]] — implementation notes empty; stub only.
- [[Screen - PI]] — not started.
- [[Screen - Jobs]] — not started.
- iOS CI secret injection — `Secrets.xcconfig` with `ESI_CLIENT_ID` must be provided in CI; not yet documented as a runbook.
- Android CI secret injection — `local.properties` with `esi.client_id` must be provided in CI; not yet set up.
- AGP 9.0 KMP migration — `com.android.library` + `org.jetbrains.kotlin.multiplatform` deprecated; must migrate to `com.android.kotlin.multiplatform.library` before AGP 10.0. See [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]].
- `@Preview` CMP 1.11 migration — `org.jetbrains.compose.ui.tooling.preview.Preview` deprecated; migrate to `androidx.compose.ui.tooling.preview.Preview` from `org.jetbrains.compose.ui:ui-tooling-preview`.

### Resolved (2026-07-09) — Stack upgrade milestone
- ~~**Kotlin 2.4.0 + CMP 1.11.1 + AGP 9.2.0 + Ktor 3.x + Koin 4.x**~~ — full stack at max compatible versions. App launches on both Android and iOS. See [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]].

### Resolved (2026-07-08) — iOS first-run build blockers
- ~~**SQLite linker error**~~ — `OTHER_LDFLAGS = "-lsqlite3"` added to both target configs. See [[Platform - iOS]].
- ~~**"Multiple commands produce Info.plist"**~~ — moved `Info.plist` to `Configuration/` (outside `PBXFileSystemSynchronizedRootGroup`). See [[Platform - iOS]].
- ~~**ESI SSO `client_id` missing**~~ — `Secrets.xcconfig` → `Info.plist` → `NSBundle` chain wired. See [[ADR-011 - Secrets via expect-actual and local.properties]] and [[Platform - iOS]].
- ~~**OAuth callback "address is invalid"**~~ — `CFBundleURLTypes` with `eveauth-podforeve` scheme added to `Info.plist`. See [[Platform - iOS]].
- ~~**`PlistSanityCheck` SIGABRT (UISceneConfigurations)**~~ — removed empty `UISceneConfigurations` dict. See [[Platform - iOS]].
- ~~**`PlistSanityCheck` SIGABRT (CADisableMinimumFrameDurationOnPhone)**~~ — added key to `Info.plist`. See [[Platform - iOS]].
- ~~**iOS app working end-to-end**~~ — SSO login, token exchange, ESI data fetch all confirmed working on iOS Simulator.

### Resolved (2026-07-08) — earlier
- ~~**Xcode project missing**~~ — `iosApp.xcodeproj` exists and is wired to KMP via `embedAndSignAppleFrameworkForXcode` build phase.
- ~~**Chucker sync**~~ — Android build confirmed working. Chucker pinned to `4.1.0` (4.3.1 compiled with Kotlin 2.3.x metadata, incompatible with KGP 2.1.x).
- ~~**CLIENT_ID in git**~~ — removed via orphan-branch force push; now injected via expect/actual + local.properties/xcconfig. See [[ADR-011 - Secrets via expect-actual and local.properties]].
- ~~**koin-compose GlobalContext on iOS**~~ — `GlobalContext.get()` is JVM-only; replaced with `KoinPlatform.getKoin()` in `App.kt` and `LoginScreen.kt`.
- ~~**`xcode-select` pointing to CLI tools only**~~ — fix: `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`.

### Resolved (2026-06-30)
- ~~EVE dev-portal registration~~ — done; callback `eveauth-podforeve://callback`.
- ~~`gradle-wrapper.jar` missing~~ — resolved by Android Studio; Gradle 8.14.5 wrapper in place.
- ~~JVM target mismatch (17 vs 21)~~ — fixed in all three build.gradle.kts.
- ~~SSO token exchange failure (basicAuth)~~ — removed; PKCE public client uses form body only.
