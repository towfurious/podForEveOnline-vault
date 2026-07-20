# Index

> Catalog of every wiki page. Maintained by the LLM agent per `[[CLAUDE]]` §5.
> One line per page: `- [[Page Title]] — one-line description`.

## Entities
- [[Character]] — the single EVE pilot the app tracks.
- [[Skill Queue]] — ordered list of training skills; source of Skills screen + Dashboard training widget.
- [[Planet]] — a PI colony; status is derived as Active / Needs Attention / Idle.
- [[Industry Job]] — active research or manufacturing job.
- [[WalletJournalEntry]] — one wallet journal transaction; `displayName` maps ESI `ref_type` → readable label.

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
- [[ADR-013 - Faction Color Themes]] — 5 M3 dark color schemes (Minmatar/Amarr/Caldari/Gallente/AMOLED); `ThemeRepository` backed by `SecureStorage`; bottom-sheet theme switcher on Dashboard.
- [[ADR-014 - Ktlint Detekt Linter Setup]] — Ktlint 1.3.1 (formatting) + Detekt 1.23.8 + `io.nlopez.compose.rules:detekt` 0.4.22 (static analysis); baselines committed; applied in root `subprojects {}`.
- [[ADR-015 - Unified Completion Notifications]] — shared `NotificationScheduler` covering skill queue, industry jobs, PI extractors; Android skill queue gets a live-countdown ForegroundService (`specialUse` type + `SCHEDULE_EXACT_ALARM`), job/extractor get plain one-shot alarms; iOS uses one-shot `UNNotificationRequest` for all three.
- [[ADR-016 - AGP KMP Library Plugin Migration]] — `shared`/`composeApp` migrated `com.android.library` → `com.android.kotlin.multiplatform.library`; `androidApp` dropped the redundant standalone Kotlin plugin for AGP's built-in support; ESI_CLIENT_ID secret injection moved from `BuildConfig` (unsupported by the new plugin) to a custom Gradle task.
- [[ADR-017 - Neon Outline Card and Icon Treatment]] — "plasma conduit" sci-fi glow treatment (from an Artifact mockup) implemented for real: new `GlowCard` component, `EveIcons` re-rendered as open strokes on their existing path data, `AppTheme.gainColor` extension (amber on Gallente). No new `Theme.kt` tokens — reuses `onPrimaryContainer` as the glow color across all 5 schemes.

## Patterns
- [[Math-Based Progress Bar]] — compute progress from start/end timestamps, tick via coroutine, never poll.

## ESI endpoints
- [[ESI Scopes MVP]] — the 5 OAuth scopes for MVP (per-endpoint pages added when wired in data-layer plan).
- [[ESI - Skills - Get Skill Queue]] — GET /v2/characters/{id}/skillqueue/ — requires esi-skills.read_skillqueue.v1.
- [[ESI - Universe - Get Type]] — GET /v3/universe/types/{type_id}/ — public; resolves skill_id → name.
- GET /v4/corporations/{id}/ — public; fetches corp name for Dashboard header.
- GET /v4/characters/{id}/skills/ — requires esi-skills.read_skills.v1; returns total_sp.
- GET /v6/characters/{id}/wallet/journal/ — requires esi-wallet.read_character_wallet.v1; last 3 entries on Dashboard.

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
- [[Guide - App Store Launch Readiness]] — master P0/P1/P2 tracker for everything needed before a real Play Store / App Store launch: release signing, CCP disclaimer, privacy policy, crash reporting, ESI etiquette compliance, CI/CD, and more.
- [[Guide - Compose Multiplatform Maturity Findings]] — living synthesis of what's rough vs. solid in CMP/KMP today, pulled from ADR-012/ADR-016/ADR-017 and the log — the project's actual point is evaluating this firsthand, not just shipping.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]] — design spec, approved 2026-04-24.

## Open threads
- [[Guide - App Store Launch Readiness]] — master tracker for everything still needed before a real store launch: release signing, CCP disclaimer, privacy policy, crash reporting, ESI etiquette compliance, CI/CD, and more. Folds in and supersedes the CI-secret-injection and AGP-9-migration bullets that used to live here standalone, plus [[ADR-015 - Unified Completion Notifications]]'s "explicitly deferred" follow-ups.
- [[ADR-015 - Unified Completion Notifications]] — iOS device/simulator behavior not yet confirmed on-device (only Android was verified, 2026-07-15).
- Per-endpoint ESI pages (list grows during data-layer plan; two added so far).
- `@Preview` CMP 1.11 migration — `org.jetbrains.compose.ui.tooling.preview.Preview` deprecated; migrate to `androidx.compose.ui.tooling.preview.Preview` from `org.jetbrains.compose.ui:ui-tooling-preview`.

### Resolved (2026-07-15) — Notification device verification
- ~~[[ADR-015 - Unified Completion Notifications]] — device-verified pending~~ — verified end-to-end on Android hardware (real login, all 3 sources: skill live-countdown, PI extractor, industry job). Found and fixed 3 real bugs in the process: wrong head-of-queue selection (bare `queuePosition == 0` instead of filtering finished entries — see [[Skill Queue]]), duplicate/orphaned skill notification (mismatched notification keys across `startForeground()`/tick/backup-alarm posts), and a channel-not-created crash reachable via an abnormal Service-start path. See `log.md` 2026-07-15 for the full trace.

### Resolved (2026-07-09) — All MVP screens implemented
- ~~[[Screen - Dashboard]] — implementation notes empty; stub only~~ — implemented: 76dp portrait, corp name, security badge, gear+logout sheet, 2-col stats (ISK+SP), training card, recent activity card.
- ~~[[Screen - PI]] — not started~~ — implemented: planet type chips (colored by type), status chips, extractor/factory rows.
- ~~[[Screen - Jobs]] — not started~~ — implemented: status chips (Active/Complete/Delivered), green progress bar for complete, "Ready to deliver" label, 0.75 opacity for non-active cards.
- ~~[[Screen - Skills]] — not started~~ — implemented: live progress bar + full queue list.

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
