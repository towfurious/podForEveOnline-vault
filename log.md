# Log

> Append-only chronological record. Entry format: `## [YYYY-MM-DD] <type> | <title>`.
> Types: `ingest | query | lint | dev | meta`.

## [2026-04-24] meta | Vault initialized
- Created vault skeleton: `CLAUDE.md`, `README.md`, `index.md`, `log.md`, `templates/`, `sources/raw/`, `wiki/{entities,concepts,decisions,patterns,esi,screens,platform,guides}/`.
- Project: EVE Online KMP + Compose Multiplatform app. Parent repo at `..` (gitignored to exclude this vault).
- Dual-role: dev-agent + LLM Wiki agent. See `[[CLAUDE]]` §1 and §4.4.
- Next proposed operations: (a) ingest spec `../docs/superpowers/specs/2026-04-24-eve-online-kmp-design.md`; (b) write Foundation implementation plan.

## [2026-06-30] meta | Close open threads: platform targets, SQLDelight versioning, Compose MP iOS audit
- **Android minSdk 28** (Android 9.0): cleartext-off defaults, clean ForegroundService story. See [[ADR-010 - Platform Targets]].
- **iOS deployment target 16.0**: covers 95%+ devices, satisfies Compose MP 1.7+ stable iOS tier. See [[ADR-010 - Platform Targets]].
- **SQLDelight versioning**: version 1 = MVP schema; `.sqm` migration files; drop-and-repopulate acceptable (pure ESI cache, no user data). See [[SQLDelight Migrations]].
- **Compose MP iOS maturity**: closed as non-issue. Compose MP 1.7+ stable covers all M3 components used in MVP. Existing caveat (LazyColumn tuning on complex lists) already noted in [[Platform - iOS]].
- Pages created: [[ADR-010 - Platform Targets]], [[SQLDelight Migrations]].
- Pages updated: [[Platform - Android]], [[Platform - iOS]], `index.md`.
- Open threads remaining: per-endpoint ESI pages, EVE dev-portal client_id registration.

## [2026-06-30] dev | Foundation scaffold — KMP project skeleton
- Code: `podForEveOnline/` — `gradle/`, `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `:shared`, `:composeApp`, `:androidApp`, `iosApp/`.
- SQLDelight schema v1 created: `AppDatabase.sq` (character, skill_queue_entry, planet, industry_job tables + named queries).
- `expect class SecureStorage` scaffolded with Android/iOS actuals (TODO stubs).
- `UiState<T>` sealed class in `shared/commonMain`.
- `App.kt` root Composable with dark M3 theme; `DashboardScreen` stub via Voyager.
- `MainActivity`, `PodForEveApplication` (Koin init stub), `AndroidManifest.xml` (permissions + OAuth intent filter + ForegroundService stub comment).
- iOS: `iOSApp.swift` + `ContentView.swift` wrapping `MainViewController` (ComposeUIViewController).
- ⚠️ `gradle-wrapper.jar` binary not committed — run `gradle wrapper --gradle-version 8.9` before first build.
- ⚠️ Xcode `.xcodeproj` not yet created — see [[Guide - Foundation Setup]] §Steps to first build.
- Wiki: [[Guide - Foundation Setup]] created and linked from `index.md`.
- Next: wire Koin modules, set up ESI Ktor client with auth interceptor, implement Auth flow.

## [2026-06-30] dev | Auth flow — OAuth2 PKCE + EVE SSO
- **New files (shared/commonMain)**:
  - `auth/EsiConfig.kt` — SSO constants; CLIENT_ID placeholder (TODO: register on EVE dev portal).
  - `auth/model/OAuthTokens.kt` — Ktor-serializable token response.
  - `auth/model/AuthState.kt` — sealed: Unauthenticated / Loading / Authenticated(characterId) / Error.
  - `auth/OAuthPkceHelper.kt` — generateVerifier, deriveChallenge (SHA-256), generateState, buildAuthUrl.
  - `auth/EsiAuthService.kt` — exchangeCode, refreshToken, extractCharacterId (JWT sub parsing).
  - `auth/AuthRepository.kt` — orchestrates flow: startLogin → url, completeLogin(callbackUrl), getValidAccessToken (auto-refresh), logout.
  - `platform/Crypto.kt` — expect fun sha256 + secureRandomBytes.
  - `di/SharedModule.kt` — Koin: HttpClient (Ktor + JSON), OAuthPkceHelper, EsiAuthService, AuthRepository.
- **New files (androidMain)**: Crypto.android.kt (MessageDigest + SecureRandom), SecureStorage.android.kt (EncryptedSharedPreferences), AppContext.kt (app context singleton), di/PlatformModule.android.kt.
- **New files (iosMain)**: Crypto.ios.kt (CC_SHA256 + SecRandomCopyBytes), SecureStorage.ios.kt (NSMutableDictionary + Security.framework Keychain), di/PlatformModule.ios.kt, KoinInit.kt (initKoin()), AuthCallbackHandler.kt (handleAuthCallback called from Swift onOpenURL).
- **Updated (androidApp)**: PodForEveApplication sets AppContext + starts Koin with [sharedModule, platformModule]; MainActivity handles onNewIntent for eve-tracker:// (launchMode=singleTask).
- **Updated (iosApp)**: iOSApp.swift calls initKoin() + onOpenURL { handleAuthCallback(url) }; AndroidManifest got launchMode=singleTask.
- **Test**: OAuthPkceTest (5 cases: verifier length/charset, challenge length, uniqueness, state format, URL params).
- **Wiki**: [[SecureStorage]] and [[OAuth2 PKCE]] pages are now implemented; no changes to the pages needed (they accurately describe what was built).
- Next: ESI data layer (Ktor auth interceptor with getValidAccessToken, first ESI call, SQLDelight driver init)

## [2026-06-30] dev | ESI data layer — SQLDelight driver, Ktor auth interceptor, SkillQueueRepository
- **DatabaseDriverFactory** (expect/actual): Android `AndroidSqliteDriver`, iOS `NativeSqliteDriver`.
- **Two Ktor HTTP clients** (named "sso" / "esi") to break circular dependency between EsiAuthService and AuthRepository. ESI client uses Ktor `Auth { bearer { loadTokens / refreshTokens } }` calling `authRepo.getValidAccessToken()`.
- **AppDatabase.sq** updated: added `skill_type` table; all upsert queries use explicit column lists (SQLDelight 2.x safe).
- **DTOs**: `EsiSkillQueueEntryDto` (skill queue response), `EsiUniverseTypeDto` (type name resolution).
- **SkillQueueEsiApi**: `fetchSkillQueue(characterId)` (auth), `fetchTypeName(typeId)` (public).
- **SkillQueueRepository**: SWR pattern — emit cached rows immediately, refresh from ESI, emit fresh. `resolveSkillName` caches type names in `skill_type` table; fallback to "Skill {id}" if offline.
- **domain/model/SkillQueueEntry**: domain model with `isTraining` computed property.
- **SharedModule** rewritten: two named HttpClients, AppDatabase, SkillQueueEsiApi, SkillQueueRepository.
- **PlatformModules** updated on both platforms to include `DatabaseDriverFactory`.
- **Added Gradle dependency**: `ktor-client-auth`, `androidx-security-crypto`.
- **Wiki**: [[ESI - Skills - Get Skill Queue]], [[ESI - Universe - Get Type]] created; `index.md` updated.
- Next: SkillsScreen ViewModel + UI (live progress bar from [[ADR-005 - Math-Based Skill Progress]]).

## [2026-06-30] dev | SkillsScreen — ViewModel, math progress bar, 4-tab navigation
- **SkillProgressCalculator** (shared/commonMain/domain/usecase/): injectable Clock, `snapshot(entry)` → `Snapshot(progress, remaining)`, `Duration.formatHms()`.
- **SkillProgressCalculatorTest** (7 cases): zero/half/one, clamping before+after, null when paused, formatHms.
- **SkillQueueViewModel** (composeApp, Voyager ScreenModel): `refreshTrigger` + `flatMapLatest` → pull-to-refresh re-runs SWR; exposes `uiState: StateFlow<UiState<List<SkillQueueEntry>>>`.
- **AppModule** (uiModule): `factory { SkillQueueViewModel(get(), get()) }`.
- **SkillsScreen.kt**: Loading shimmer, Success (hero progress + queue list), Error (inline + retry), Paused banner, Empty state. Uses `PullToRefreshBox` (M3 experimental).
- **ActiveSkillProgressSection**: `produceState` ticks every 1 s. No network per tick per [[ADR-005 - Math-Based Skill Progress]].
- **SkillQueueRow**: ticks every 60 s (minute-level sufficient for queue tail).
- **Shimmer.kt**: `Modifier.shimmer()` via infinite alpha animation — no extra library per [[ADR-009 - UiState Sealed Class with Shimmer]].
- **App.kt**: `TabNavigator` + M3 `NavigationBar` with 4 tabs (Dashboard, Skills, PI stub, Jobs stub).
- **KoinInit split**: `composeApp/iosMain/KoinInit.kt` initializes all 3 modules (shared + platform + ui). `shared/iosMain/KoinInit.kt` emptied. `PodForEveApplication` updated to include `uiModule`.
- **composeApp/build.gradle.kts**: `api(projects.shared)` + `export(projects.shared)` on iOS frameworks — single `composeApp.framework` for Xcode.
- Wiki: [[Screen - Skills]] implementation notes filled in.
- Next: SkillQueueViewModelTest or Dashboard screen (ISK + training widget).

## [2026-06-30] dev | Login flow — LoginScreen, UrlLauncher, platform actuals
- **LoginScreen.kt** (composeApp/commonMain): `koinInject<AuthRepository>()`, `collectAsState()`, button calls `startLogin() → urlLauncher(url)`, `CircularProgressIndicator` while loading, inline error text.
- **platform/UrlLauncher.kt** (expect): `@Composable expect fun rememberUrlLauncher(): (String) -> Unit`.
- **UrlLauncher.android.kt**: Chrome Custom Tabs via `CustomTabsIntent.Builder().build().launchUrl(context, Uri.parse(url))`.
- **UrlLauncher.ios.kt**: `UIApplication.sharedApplication.openURL(nsUrl, options: emptyMap, completionHandler: null)`.
- **iOSApp.swift** updated: `init() { KoinInitKt.initKoin() }` + `.onOpenURL { AuthCallbackHandlerKt.handleAuthCallback(url:) }`.
- Note: `startLogin()` is a regular (non-suspend) function; wrapped in `scope.launch` in LoginScreen only for lifecycle safety.
- Wiki: no new pages; LoginScreen implementation matches [[Screen - Dashboard]] pattern — implementation notes deferred.

## [2026-06-30] dev | Auth gate, Android resources, browser dep, Info.plist
- **App.kt** rewritten: auth gate — `Loading → SplashScreen`, `Unauthenticated/Error → LoginScreen`, `Authenticated → MainApp`. `LaunchedEffect` triggers `getValidAccessToken()` on startup to restore session from stored refresh token.
- **Android resources created**: `res/values/themes.xml` (`Theme.PodForEve` extends `AppCompat.DayNight.NoActionBar`, black window bg); `res/values/strings.xml` (`app_name`); `res/drawable/ic_launcher_background.xml` (dark `#0D1117`); `res/drawable/ic_launcher_foreground.xml` (vector "P" in EVE-blue); `res/mipmap-anydpi-v26/ic_launcher{,_round}.xml` (adaptive icon).
- **AndroidManifest.xml**: fixed theme ref from `@style/Theme.AppCompat` → `@style/Theme.PodForEve`.
- **proguard-rules.pro**: keeps for Ktor, SQLDelight, Koin, kotlinx-serialization.
- **libs.versions.toml**: added `androidx-browser = "1.8.0"` version + library entry.
- **androidApp/build.gradle.kts**: added `libs.androidx.browser` dependency (required for `CustomTabsIntent`).
- **iosApp/iosApp/Info.plist**: created with `CFBundleURLTypes` containing `eve-tracker` scheme — routes OAuth callback to app.
- ⚠️ `gradle-wrapper.jar` still missing — user runs `gradle wrapper --gradle-version 8.9` before first build.
- ⚠️ `EsiConfig.CLIENT_ID = "YOUR_EVE_CLIENT_ID"` — placeholder until dev-portal registration.
- ⚠️ Xcode `.xcodeproj` does not yet exist — iOS cannot build without it.
- Wiki: open threads updated in `index.md`.

## [2026-04-24] ingest | EVE Online KMP Design Spec
- Source raw: `sources/raw/2026-04-24 EVE Online KMP Design Spec.md` (snapshot of approved spec at `~/docs/superpowers/specs/2026-04-24-eve-online-kmp-design.md`).
- Source page: [[Source - 2026-04-24 - EVE Online KMP Design Spec]].
- Pages created (25):
    - Entities: [[Character]], [[Skill Queue]], [[Planet]], [[Industry Job]].
    - Concepts: [[OAuth2 PKCE]], [[UiState]], [[SecureStorage]], [[Stale-While-Revalidate Cache]].
    - Decisions: [[ADR-001 - KMP Compose Multiplatform]], [[ADR-002 - Material 3 Dark Default]], [[ADR-003 - Voyager and Bottom Navigation]], [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]], [[ADR-005 - Math-Based Skill Progress]], [[ADR-006 - Android Foreground Service]], [[ADR-007 - iOS Local Notifications]], [[ADR-008 - OAuth2 PKCE via System Browser]], [[ADR-009 - UiState Sealed Class with Shimmer]].
    - Pattern: [[Math-Based Progress Bar]].
    - Screens: [[Screen - Dashboard]], [[Screen - Skills]], [[Screen - PI]], [[Screen - Jobs]].
    - Platform: [[Platform - Android]], [[Platform - iOS]].
    - ESI: [[ESI Scopes MVP]].
- `[[index]]` rewritten with all sections populated.
- Open threads logged: per-endpoint ESI pages pending, dev-portal registration status, min API / deployment target, SQLDelight versioning strategy, Compose MP iOS M3 audit.
- Key changes: all nine MVP architectural decisions captured as accepted ADRs; all four MVP screens have living pages ready to accept implementation notes during dev.

## [2026-07-08] dev | UI polish — safe areas, portrait loading, timer format
- **White safe areas**: SwiftUI `ZStack` with `Color(20,18,24).ignoresSafeArea()` behind `ContentView` closes status-bar and home-indicator gaps.
- **NavigationBar color**: `containerColor = MaterialTheme.colorScheme.surface` — tab bar now matches app background, bottom strip seamless.
- **Portrait loading on iOS**: `coil-network-ktor2` added to `iosMain` deps. `initCoil()` in `MainViewController.kt` registers `KtorNetworkFetcherFactory(HttpClient(Darwin))` as Coil singleton before first frame. Called from Swift `init()` after `doInitKoin()`. Note: `MainViewControllerKt.initCoil()` → ObjC mangling → `doInitCoil()` in generated header.
- **Timer format**: Added `formatDhm()` alongside `formatHms()`. Queue rows and Jobs screen show `10d 3h 18m`; active-skill header keeps `h m s` precision. EVE-standard convention (days when > 24 h).
- **Training card overflow**: `Modifier.weight(1f)` + `maxLines=1` + `TextOverflow.Ellipsis` on skill name text — timer no longer overflows the row.
- Wiki: no new pages; changes are implementation detail.

## [2026-07-08] dev | iOS first-run build — 6 blockers resolved, app working end-to-end
- **SQLite linker**: added `OTHER_LDFLAGS = "-lsqlite3"` to both Debug/Release target build configs in `project.pbxproj`. Root cause: static KMP framework doesn't propagate C-interop system lib dependencies.
- **"Multiple commands produce Info.plist"**: moved `Info.plist` to `iosApp/iosApp/Configuration/` (outside `PBXFileSystemSynchronizedRootGroup` auto-synced dir). Set `GENERATE_INFOPLIST_FILE = NO`, `INFOPLIST_FILE = Configuration/Info.plist`.
- **ESI `client_id` missing**: wired `Configuration/Secrets.xcconfig` (gitignored, `ESI_CLIENT_ID = …`) as `baseConfigurationReference` in pbxproj. `Info.plist` has `$(ESI_CLIENT_ID)` which Xcode expands at build time.
- **OAuth callback "address is invalid"**: registered `eveauth-podforeve` scheme in `CFBundleURLTypes` in `Info.plist`.
- **`PlistSanityCheck` SIGABRT ×2**:
  - Remove empty `UISceneConfigurations: {}` from `UIApplicationSceneManifest`.
  - Add `CADisableMinimumFrameDurationOnPhone = true`. Confirmed these are the only two checks in CMP 1.7.3 via binary search (UTF-16 strings) of the linked dylib.
- **ObjC name mangling**: `initKoin()` → `KoinInitKt.doInitKoin()` in Swift (`init`-prefix gets `do` prepended by Kotlin/Native codegen).
- **Result**: iOS Simulator app launches, completes EVE SSO, fetches ESI data (skills, wallet, skill queue) — all confirmed in console logs.
- Wiki: [[Platform - iOS]] major update (new "Xcode build gotchas" section); `index.md` open threads updated.

## [2026-07-08] meta | PostCompact vault auto-reload hook
- Added `PostCompact` hook to `.claude/settings.local.json`: reads `index.md` + tail of `log.md` and injects content as `additionalContext` after every context compaction.
- Created `CLAUDE.md` at `PodForEve/` root: loads at session start, instructs agent to read vault.
- Net effect: vault content is automatically present in context after compaction without manual action.

## [2026-07-08] dev | Kotlin 2.1.21 + CMP 1.7.3 upgrade; Chucker 4.1.0 pin
- **Kotlin**: `2.0.21` → `2.1.21`. Required for Xcode 16.3+ support (kdoctor warning).
- **Compose Multiplatform**: `1.6.11` → `1.7.3`. Paired release for Kotlin 2.1.x.
- **Chucker**: Downgraded from `4.3.1` → `4.1.0`. Chucker 4.3.1 was compiled with Kotlin 2.3.x metadata (binary version 2.3.0); KGP 2.1.21 only understands up to 2.1.0, causing `IllegalArgumentException: Unsupported metadata version` at compile time. `4.1.0` is compiled with compatible metadata.
- `android-compileSdk` remains `36` (required by Chucker 4.1.0+).
- File: `gradle/libs.versions.toml`.

## [2026-07-08] dev | Secrets removal — expect/actual esiClientId(), local.properties, force push
- `EsiConfig.CLIENT_ID` was a hardcoded string literal committed to git history.
- Rewrote as `expect fun esiClientId()` with platform actuals:
  - Android: `BuildConfig.ESI_CLIENT_ID` populated from `local.properties` via `buildConfigField` in `shared/build.gradle.kts`.
  - iOS: `NSBundle.mainBundle.objectForInfoDictionaryKey("ESIClientID")` (xcconfig → Info.plist).
- Added `import java.util.Properties` to `shared/build.gradle.kts` (Kotlin Gradle DSL requires explicit import).
- Added `buildFeatures { buildConfig = true }` to shared module's android block.
- Removed secret from git history via orphan branch force push.
- Wiki: [[ADR-011 - Secrets via expect-actual and local.properties]] created.

## [2026-07-08] dev | Skill queue — filter completed entries, fix row numbering
- **Root cause**: ESI `GET /v2/characters/{character_id}/skillqueue/` returns entries with `finish_date` in the past (already completed) without immediately removing them. `entries.first()` was selecting a finished skill.
- **Fix 1 — domain model**: added `hasFinished(nowEpochSeconds: Long): Boolean` to `SkillQueueEntry`. Logic: `finishDate != null && finishDate <= now`.
- **Fix 2 — SkillsScreen**: `filterNot { it.hasFinished(now) }` before selecting head. Uses `Clock.System.now().epochSeconds`.
- **Fix 3 — DashboardViewModel**: same filter applied when selecting `activeSkill` for training widget.
- **Fix 4 — row numbering**: after filtering, `queue_position` values are non-contiguous. Added `displayPosition: Int` parameter to `SkillQueueRow`; passed `index + 2` from `itemsIndexed`.
- Wiki: [[Screen - Skills]], [[Skill Queue]], [[ESI - Skills - Get Skill Queue]] updated with gotcha and fix.

## [2026-07-08] dev | Adaptive icon inset fix
- Problem: 48dp PNG foreground scaled ×2.25 to fill 108dp adaptive canvas, extending beyond the 66dp safe zone and appearing cropped on device.
- Fix: wrapped foreground drawable in `<inset android:inset="28%"/>` in `ic_launcher.xml` and `ic_launcher_round.xml`. Reduces effective display size to ~48dp (1:1 scale), fitting within safe zone.
- New pod-shaped vector icon created as `ic_launcher_foreground_pod.xml` (EVE capsule design) but not yet activated as adaptive foreground.

## [2026-07-08] dev | App icons — LANCZOS downscale from 512px source + monochrome
- **Source**: `podForEveOnline/02_icon/ic_launcher_APP.png` (512×512 rendered PNG).
- **Android legacy icons** (LANCZOS downscale via PIL): `mipmap-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/ic_launcher{,_round}.png` at 48/72/96/144/192px.
- **Android adaptive foreground** (108dp canvas sizes): `mipmap-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/ic_launcher_foreground.png` at 108/162/216/324/432px. Removed `<inset>` wrapper from `ic_launcher{,_round}.xml` — foreground now proper canvas size.
- **Monochrome icon**: `drawable/ic_launcher_monochrome.png` — pixels with color distance > 28 from background `#22112F` become white, others transparent. Used for Android 13+ themed icons.
- **iOS AppIcon**: `AppIcon_1024.png` (sips from 512px source) for light+dark slots; `AppIcon_1024_tinted.png` (grayscale) for tinted slot. `Contents.json` updated with `filename` fields for all 3 universal slots.
- **`ic_launcher.xml` / `ic_launcher_round.xml`**: removed inset wrapper, added `<monochrome>` element pointing to `@drawable/ic_launcher_monochrome`.

## [2026-07-08] dev | Warp-in splash screen — vtracer-traced bezier paths
- **Concept**: pod blob drops from top-right diagonally to center (warp streak effect), then 3 dark port-holes spring-pop in one-by-one with 130ms stagger — EVE hyperspace exit aesthetic.
- **Path accuracy**: `ic_launcher_APP.png` → white-on-black silhouette → `vtracer` → SVG → parsed to Kotlin `Path` API. Blob = 96 `cubicTo` calls traced from actual icon contour; 3 hole paths traced from actual oval ports.
- **`PodSplashScreen.kt`** (`composeApp/commonMain/…/ui/component/`):
  - `Animatable` `shellOffset`/`shellAlpha` for blob warp-in (420 ms `EaseOutExpo` tween).
  - `hole1/2/3Scale` spring-pop (dampingRatio 0.60, stiffness 480/400/340).
  - `drawWarpStreak`: gradient `Color(0x00F5D060)` → `0xFFE8CC60` line, strokeWidth lerp 22→3.
  - `drawPodBlob`: 5-stop radial gradient (yellow → orange → red → purple → black), rim specular rect, 15 star sparkles at SVG-traced positions, 3 hole paths scaled via `withTransform { scale(s, s, pivot) }` inside `clipPath`.
  - Stars: real positions from vtracer subpaths (e.g. `0.607, 0.185`; `0.822, 0.439`).
  - Hole pivots (for scale transform): H1=(0.500, 0.390), H2=(0.400, 0.547), H3=(0.603, 0.653).
- **App.kt**: `SplashScreen()` composable → `PodSplashScreen()`.
- Build: `./gradlew :composeApp:compileDebugKotlinAndroid` — BUILD SUCCESSFUL.
- Wiki: no new pages; implementation detail only.

## [2026-07-08] dev | iOS build — GlobalContext fix, NSBundle import, xcode-select
- **`GlobalContext` JVM-only**: `org.koin.core.context.GlobalContext` does not exist in Kotlin/Native. `App.kt` and `LoginScreen.kt` were using it; replaced with `org.koin.mp.KoinPlatform.getKoin()`. `AuthCallbackHandler.kt` (shared/iosMain) was already correct.
- **Missing `NSBundle` import**: `EsiClientId.ios.kt` used `NSBundle.mainBundle` without `import platform.Foundation.NSBundle`. Added.
- **Compilation verified**: `./gradlew :shared:compileKotlinIosSimulatorArm64` — BUILD SUCCESSFUL. `./gradlew :composeApp:compileKotlinIosSimulatorArm64` — BUILD SUCCESSFUL after the two fixes above.
- **Link step blocked by `xcode-select`**: `./gradlew :composeApp:linkDebugFrameworkIosSimulatorArm64` fails with `xcrun exit 72` when `xcode-select -p` points to `/Library/Developer/CommandLineTools` instead of Xcode.app. Fix: `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`. (Code issue resolved; environment fix pending on user machine.)
- Wiki: [[Platform - iOS]] updated with compilation gotchas and Xcode project status.

## [2026-06-30] dev | First Android build — Gradle fixes, OAuth end-to-end, Chucker

### Gradle / dependency fixes
- `coil3` downgraded to `3.0.4` (stable; `3.0.0-rc02` no longer on Maven Central).
- `koin` pinned to `3.5.6` (`3.6.0` not released; separate `koin-compose` version entry removed).
- `koin-compose` (`io.insert-koin:koin-compose`) removed from `composeApp` entirely — in Koin 3.x this artifact has no native (iOS) metadata and breaks KMP compilation. Replaced with `koin-core` + direct `GlobalContext.get().get<T>()` calls in Composables (`App.kt`, `LoginScreen.kt`).
- JetBrains Space maven repo added to `settings.gradle.kts` (required for Voyager `1.1.0-alpha04`).
- `kotlinx-datetime` added explicitly to `composeApp` commonMain (was `implementation` not `api` in shared, so not transitive).
- `androidx.browser` moved from `androidApp` to `composeApp` androidMain (where `UrlLauncher.android.kt` lives).
- `coil3-network-ktor` removed from `composeApp` (unused, caused resolution error).

### Build toolchain
- AGP upgraded to `8.13.2` (user action; latest recommended by Android Studio).
- Gradle wrapper updated to `8.14.5` (user action in `gradle-wrapper.properties`).
- Java target: `VERSION_17` → `VERSION_21` in all three `build.gradle.kts` files (`androidApp`, `shared`, `composeApp`) — JVM installed is 21, mismatch caused KotlinJvmTarget error.
- `adb` and `emulator` PATH entries added to `~/.zshrc` (user action; provided exact export lines).

### Kotlin compilation fixes
- **`SkillQueueRepository.kt`**: `resolveSkillName()` is a `suspend` function but was called inside `db.transaction{}` (plain lambda). Fixed: resolve all names into a `Map<Long, String>` before entering the transaction; reference the map inside.
- **`SkillsScreen.kt`**: `PullToRefreshBox` not available in Compose Multiplatform 1.6.11 commonMain. Replaced with plain `Box`; removed `isRefreshing` usage.
- **`AppContext.kt` (androidMain)**: was `internal object AppContext` — inaccessible from `androidApp` module. Removed `internal` modifier.

### OAuth2 / EVE SSO
- `EsiConfig.CLIENT_ID` set to `e82d24f308f2419e8840765769373c0f` (registered on EVE dev portal).
- Callback URL corrected: EVE portal requires scheme starting with `eveauth`. Final: `eveauth-podforeve://callback`. Updated in `EsiConfig.kt`, `AndroidManifest.xml` (intent-filter scheme), `MainActivity.kt` (startsWith check), `Info.plist` (CFBundleURLSchemes).
- `EsiAuthService`: removed `basicAuth(username = clientId, password = "")` from both `exchangeCode` and `refreshToken`. EVE SSO PKCE public client requires `client_id` ONLY in the form body — Basic auth header causes token exchange to fail with "Fields [access_token…] missing at path $".
- Scope `esi-wallet.read_character_wallets.v1` corrected to `esi-wallet.read_character_wallet.v1` (singular). Added `publicData` scope.
- **Result: SSO login confirmed working end-to-end** — login page appeared, user authenticated, tokens exchanged successfully.

### HTTP client architecture refactor (for Chucker)
- HTTP clients (`ssoClient` named "sso", `esiClient` named "esi") moved out of `SharedModule` into platform-specific modules so each platform can use its own engine.
- `SharedModule.kt` now only declares non-network singletons; HTTP clients are resolved by qualifier.
- `PlatformModule.android.kt`: creates both clients with `HttpClient(OkHttp) { engine { preconfigured = get<OkHttpClient>() } }`.
- `PlatformModule.ios.kt`: creates both clients with `HttpClient(Darwin) { … }`.

### Chucker integration
- Chucker `4.0.0` added: `debugImplementation(libs.chucker)`, `releaseImplementation(libs.chucker.no.op)` in `androidApp/build.gradle.kts`.
- `ChuckerModule.kt` created in `androidApp`: provides `OkHttpClient` singleton with `ChuckerInterceptor` attached.
- `PodForEveApplication`: `chuckerModule` added to `startKoin { modules(…) }`.
- Both Android Ktor clients (`sso` + `esi`) use `preconfigured = get<OkHttpClient>()` — all traffic visible in the Chucker notification shade overlay.
- Version entry and library aliases added to `libs.versions.toml`; Chucker served from Maven Central (not JitPack — no extra repo needed).

### DI research note
- Surveyed KMP DI landscape (2025): Koin remains the pragmatic default for KMP. Kodein largely abandoned. Kotlin-inject promising but immature for large projects. No change to stack.

## [2026-07-09] dev | PodSplashScreen — precise capsule trace + warp mini-capsules
- **Blob shape**: hand-approximation replaced with 188-curve trace from Vectorizer.io (SVG `FreeSample-Vectorizer-io-icon.svg`). Centroid normalised: `ox = cx − 0.5002·sc`, `oy = cy − 0.4990·sc`.
- **Mini-capsules**: the three dark patches on the gradient surface are not holes but the same capsule shape scaled down (7%, 14%, 12% of the main blob). Implemented via `buildMiniBlob()` — calls `buildBlobPath` with a smaller `sc`.
- **Positions** extracted to 9 constants `M1_CX/CY/F`, `M2_*`, `M3_*` — single source of truth, no path/pivot drift.
- **Mini-capsule animation**: pop-scale (EaseOutBack) replaced with diagonal warp-in (COS45/SIN45), `warpMag = sc * 0.40`. 110 ms stagger. Yellow streak on each (stroke width `lerp(sc*0.014, sc*0.002, 1−warp)`).
- **Final sequence**: main capsule warp 380 ms → mini-1 200 ms → +110 ms mini-2 → +110 ms mini-3 → `onFinished()` after 310 ms.
- File: `composeApp/src/commonMain/.../ui/component/PodSplashScreen.kt`.

## [2026-07-09] dev | Kotlin 2.2.21 + CMP 1.9.3 upgrade + haze blur tab bar
- **Kotlin**: `2.1.21` → `2.2.21`. **CMP**: `1.7.3` → `1.9.3`. Required to match haze 1.7.1 ABI.
- **haze 1.7.1** added (`dev.chrisbanes.haze:haze`). API changed: `Modifier.haze()` → `hazeSource()`, `hazeChild()` → `hazeEffect()`.
- **`kotlinx-datetime`**: `0.6.1` → `0.7.1` (CMP 1.9.3 pulls 0.7.1 transitively; aligned explicit version to avoid split). In 0.7.1 `Clock` and `Instant` moved to `kotlin.time` package — no longer in `kotlinx.datetime`.
- **Import migration**: all `import kotlinx.datetime.Clock` → `import kotlin.time.Clock`, `import kotlinx.datetime.Instant` → `import kotlin.time.Instant` across 10 files (both modules).
- **`@file:OptIn(kotlin.time.ExperimentalTime::class)`** added to every file using `Clock.System`, `Instant.fromEpochSeconds`, or `Instant.parse` (7 files in shared, 4 in composeApp).
- **Blur tab bar**: `MainApp()` creates `HazeState`; content box has `hazeSource(hazeState)`; `PodNavBar` surface has `hazeEffect(hazeState, HazeStyle(blurRadius=24.dp))`. Alpha reduced to 0.75f, `tonalElevation=0`.
- **Build verified**: `:composeApp:compileKotlinIosSimulatorArm64` and `:composeApp:compileDebugKotlinAndroid` both `BUILD SUCCESSFUL` with zero errors.
- Files: `gradle/libs.versions.toml`, `composeApp/build.gradle.kts`, `App.kt`, all screen files + shared repository/calculator files.

## [2026-07-09] dev | Full stack upgrade — Kotlin 2.4.0 + CMP 1.11.1 + AGP 9.2.0 + Ktor 3.x + Gradle 9.4.1

- **Kotlin**: `2.2.21` → `2.4.0`. **CMP**: `1.9.3` → `1.11.1`. **AGP**: `8.13.2` → `9.2.0`. **Gradle**: `8.14.5` → `9.4.1`.
- **Ktor**: `2.x` → `3.5.1`. **Koin**: `3.x` → `4.1.1`. **kotlinx-datetime**: `0.7.1` → `0.8.0`. **coil3**: `3.x` → `3.5.0`. **haze**: `1.7.1` → `1.7.2`.
- **AGP 9.0 + KMP**: `com.android.library` + `org.jetbrains.kotlin.multiplatform` are now deprecated together (not an error yet); compat kept via `android.builtInKotlin=false` + `android.newDsl=false` in `gradle.properties`. Migration to `com.android.kotlin.multiplatform.library` deferred.
- **`kotlin.native.useEmbeddableCompilerJar` / `kotlin.mpp.androidGradlePluginCompatibility.nowarn`**: removed from `gradle.properties` — both deprecated and no longer needed in K2.4+.
- **CMP 1.11.1 dropped iosX64**: removed `iosX64()` from both `shared/build.gradle.kts` and `composeApp/build.gradle.kts`.
- **CMP `compose.*` accessors**: kept (string-literal alternatives without BOM don't resolve). `compose.components.uiToolingPreview` still used; deprecation warning is non-blocking.
- **Coil / Ktor 3**: `coil-network-ktor2` → `coil-network-ktor3`; import in `MainViewController.kt` updated from `coil3.network.ktor2` → `coil3.network.ktor3`; `@file:OptIn(coil3.annotation.ExperimentalCoilApi::class)` added.
- **`androidApp/build.gradle.kts`**: removed `alias(libs.plugins.kotlin.android)` — AGP 9.0 has built-in Kotlin support, plugin causes an error.
- **Build verified**: `:androidApp:assembleDebug`, `:composeApp:compileKotlinIosSimulatorArm64`, `:composeApp:linkDebugFrameworkIosSimulatorArm64` all `BUILD SUCCESSFUL`.
- **Open thread**: migrate to `com.android.kotlin.multiplatform.library` plugin when JetBrains publishes an official guide (tracking deprecation warning will become an error in AGP 10.0).

## [2026-07-09] dev | Tab bar — equal pill widths, frosted glass, haze clip fix
- **`kotlin.android` plugin**: accidentally removed from `androidApp/build.gradle.kts` during AGP 9 upgrade. `android.builtInKotlin=false` (needed for KMP modules) disables built-in Kotlin compilation globally, so `androidApp` stopped compiling `.kt` files — APK still assembled but `PodForEveApplication` was absent from DEX → `ClassNotFoundException` on launch. Plugin restored.
- **haze clip**: `hazeEffect` draws blur before `Surface` applies its `shape` clip, so corners remained square. Added `.clip(RoundedCornerShape(50))` before `hazeEffect` in the Surface modifier chain.
- **Equal pill widths**: `PodNavItem` gained a `modifier: Modifier` parameter; Row passes `Modifier.weight(1f)` — all 4 slots equal width regardless of label length. `Arrangement.SpaceEvenly` removed (weight handles distribution).
- **Frosted glass**: `Surface alpha` `0.75f` → `0.1f` — near-pure blur, minimal background colour overlay.
- **Icons**: `size(32.dp)` → `36.dp`.

## [2026-07-09] dev | PodNavBar — floating island tab bar
- Removed full-width `Surface` with `tonalElevation=3`. Replaced with `Box(padding=horizontal 20dp)` → `Surface(shape=RoundedCornerShape(50), shadowElevation=8dp)`.
- Tab bar now floats above content as a pill island (Instagram-style), detached from screen edges.
- Inactive items: `pillColor = Color.Transparent` (was `surface`) — item background invisible against the island.
- File: `composeApp/src/commonMain/.../App.kt` — `PodNavBar()` and `PodNavItem.pillColor` only.

## [2026-07-09] dev | PI + Jobs screen polish
- **PI**: replaced inline `"Type · Level X"` text with `PlanetTypeChip` — a colored pill chip per EVE planet type (barren/plasma/storm/oceanic/temperate/lava/ice/gas). Level remains as plain text beside it.
- **Jobs**: added `JobStatusChip` — Active (gold) / Complete (green) / Delivered (gray). Completed jobs: green progress bar, "Ready to deliver" label, 0.75 opacity. Previously showed "0m".
- Wiki: [[Screen - PI]], [[Screen - Jobs]] — implementation notes filled.
- BUILD SUCCESSFUL (compileDebugKotlinAndroid, 0 errors).

## [2026-07-09] dev | Dashboard enrichment — security status, corp, SP, wallet journal, logout
- **Code**: `shared/` — `EsiCorporationInfoDto`, `EsiCharacterSkillsDto`, `EsiWalletJournalEntryDto` (new DTOs); `WalletJournalEntry` domain model; `CharacterInfo` +3 fields (defaults 0.0/""/0L); `CharacterEsiApi` +3 methods; `CharacterRepository` — parallel ESI fetch (info+wallet+skills simultaneously, corp waits on info), `observeWalletJournal()` (emits empty first).
- **Code**: `composeApp/` — `EveIcons.Settings` gear icon (24×24, EvenOdd); `DashboardViewModel` — 3-flow combine, `fun logout()`; `DashboardScreen` — 76 dp portrait, corp+security badge, 2-col stats (ISK+SP), gear button → ModalBottomSheet logout, recent activity card (3 journal rows, relative time).
- No DB migration; no new ESI scopes (wallet+skills scopes already granted).
- Wiki: [[Screen - Dashboard]] — implementation notes filled.
- BUILD SUCCESSFUL (compileDebugKotlinAndroid, 0 errors).

## [2026-07-09] dev | UI screenshots v1 — all 4 screens
- First live screenshots from a real device (iOS, dark theme).
- `screen-pi-2026-07-09.png` — 5 planets Yahyerer system; Barren/Plasma/Storm/Oceanic chips correct; Attention ×2, Idle ×3.
- `screen-jobs-2026-07-09.png` — Fusion S Blueprint TE Research 10 runs; Complete chip green, 100% progress bar, "Ready to deliver".
- `screen-dashboard-2026-07-09.png` — ToWFurious / Republic University / +0.7 sec; ISK 99.97M + SP 32.59M; Training + Recent activity (2 entries).
- `screen-skills-2026-07-09.png` — Coherent Ore Processing Lv4 hero bar; queue of 25 skills; Total 55d 5h 4m.
- Wiki: [[Screen - PI]], [[Screen - Jobs]], [[Screen - Dashboard]], [[Screen - Skills]] — Current UI sections filled.

## [2026-07-09] dev | Error states — user-friendly messages, timeouts, auto-logout
- **EsiErrorMapper** (`shared/commonMain`): maps exceptions to readable strings — auth (401/403), timeout, server error (5xx), network error, generic fallback.
- **HttpTimeout plugin** added to both Android + iOS Ktor clients: 30s request / 15s connect.
- **Auto-logout on token expiry**: `refreshTokens` lambda in both platform modules now calls `authRepo.logout()` when `getValidAccessToken()` returns null — triggers `AuthState.Unauthenticated` → App.kt routes to LoginScreen automatically.
- **Shared `ErrorState` composable** (`composeApp/commonMain/ui/component/ErrorState.kt`): centered message + "Try again" OutlinedButton; replaces duplicate error composables in all 4 screens.
- **Repositories updated**: `CharacterRepository`, `SkillQueueRepository`, `PlanetRepository`, `IndustryJobRepository` — now call `EsiErrorMapper.userMessage(e)` instead of raw exception message.
- Commit: `f77fc46`. BUILD SUCCESSFUL (compileDebugKotlinAndroid + compileKotlinIosSimulatorArm64, 0 errors).

## [2026-07-10] dev | Faction color themes + in-app theme switcher

- **`AppTheme` enum** (5 entries): `EMBER` ("Minmatar"), `AMARR`, `CALDARI`, `GALLENTE`, `AMOLED`. `toColorScheme()` maps to M3 `darkColorScheme`; `previewColor` exposes the dot color for the picker.
- **Color schemes**: Minmatar = existing ember red; Amarr = gold `#D4A820` + purple bg `#0A0608`; Caldari = ice blue `#4888E0` + dark navy `#06080E`; Gallente = emerald `#30B858` + near-black green `#060A07`; AMOLED = electric blue `#4090FF` + pure black `#000000`.
- **`ThemeRepository`** (new, `composeApp/commonMain`): wraps `MutableStateFlow<AppTheme>` + `SecureStorage.THEME` key — persists selected theme across restarts. `KoinPlatform.getKoin().get()` used in Composables (no `koin-compose` dependency needed).
- **`SecureStorage`**: `THEME = "app.theme"` key added to `SecureStorageKeys`.
- **`AppModule`**: `single { ThemeRepository(get()) }` added to `uiModule`.
- **`App.kt`**: `MaterialTheme(colorScheme = appTheme.toColorScheme())` replaces hardcoded `EmberColorScheme`.
- **`DashboardScreen.kt`**: gear button opens single `ModalBottomSheet` with two-page content — menu page (`Appearance` row + `Log out` row) and Appearance page (back button `‹` + `ThemeRow` per theme with 28dp dot + "✓" checkmark). `showAppearance` boolean controls which page; swipe-down on Appearance page returns to menu.
- Commits: `fe0de44`.

## [2026-07-10] dev | Ktlint + Detekt + compose-rules linter setup

- **Ktlint** (`org.jlleitschuh.gradle.ktlint` 12.1.2, formatter 1.3.1): formatting enforcer. Applied in root `subprojects {}`. Config: `.editorconfig` (`intellij_idea` style, 140 char limit, `ktlint_standard_function-naming = disabled` for Composables).
- **Detekt** (`io.gitlab.arturbosch.detekt` 1.23.8): static analysis. `buildUponDefaultConfig = true`. Config: `config/detekt/detekt.yml` — `formatting` section removed (Ktlint owns formatting), Compose-friendly overrides (`LongMethod.threshold=80`, `LongParameterList.functionThreshold=8` + `ignoreAnnotated: ["Composable"]`, `MagicNumber.active=false`, `FunctionNaming.ignoreAnnotated: ["Composable"]`, `MaxLineLength: 140`).
- **compose-rules** (`io.nlopez.compose.rules:detekt` 0.4.22): Compose-specific Detekt rules. Note: artifact group is `io.nlopez.compose.rules` (NOT `io.github.mrmans0n` — that group does not exist on Maven Central).
- **Baselines**: `./gradlew detektBaseline` run once — `composeApp/detekt-baseline.xml` + `shared/detekt-baseline.xml` committed. Only new issues fail CI.
- **Generated code exclusion**: `ktlint filter { exclude { entry -> entry.file.toString().contains("/build/generated/") } }` — SQLDelight and Compose resource generated files skipped.
- **`ThemeRepository` rename**: backing property `_theme` → `_themeFlow` to satisfy `property-naming` rule (backing property must match the public property name).
- **`shared/iosMain/KoinInit.kt` deleted**: was an empty placeholder (real init is in `composeApp/iosMain/KoinInit.kt`).
- **`ktlintFormat` applied**: 45 files auto-reformatted (whitespace / trailing commas / import ordering). Zero logic changes.
- Commits: `643941b` (linter config), `fd8a514` (auto-format).

## [2026-07-09] meta | Wiki language set to English-only
- All wiki content must now be written in English (user preference).
- Added language rule to CLAUDE.md preamble.
- Updated §4.5 (UI versioning) from Russian to English.
- Updated version tables in Current UI sections of all 4 screen pages.
- Past Russian log entries translated to English retroactively (user request).

## [2026-07-11] dev | PI compact card design + progress bar fixes
- **Code — PiScreen.kt**: card padding 20/20dp → 14/12dp; planet name 22sp → 17sp; countdown 26sp → 20sp; divider padding 16/14 → 10/8dp; colony spacing 14dp → 8dp; factory icon 15dp → 13dp; factory dots 11dp/6dp-gap → 8dp/5dp-gap; storage icon/labels proportionally reduced; storage bar 6dp → 5dp; skeleton height 130dp → 110dp.
- **Code — JobsScreen.kt**: status chip moved to top-right (next to activity name); runs count moved to bottom-right; bar height 6dp → 5dp.
- **Code — SkillProgressBar.kt** (`GradientProgressBar`): track color `surfaceContainerHighest` → `onSurface.copy(alpha=0.15f)` (always visible over card backgrounds); hero bar (`ActiveSkillProgressSection`) 8dp → 6dp for consistency.
- **Root cause of invisible progress bar track**: `surfaceContainerHighest` matched `surfaceContainerLow` (card background) in dark M3 — track was invisible. `onSurface.copy(alpha=0.15f)` semi-transparent white is always visible.
- Wiki: [[Screen - PI]], [[Screen - Jobs]], [[Screen - Skills]] — implementation notes updated (pending).

## [2026-07-11] dev | PI volume table + storage logic fixes
- **Code — PiVolumeTable.kt**: P0 volume 0.01 → **0.005** m³/u (EVERef SDE verified); P1 0.1 → **0.19** m³/u; added type 2308 (Suspended Plasma) to P0 list (was missing → 20× overflow on Storm planet); added P1 types 3645, 3683; added P2 types 2463, 3828, 9838; fallback 0.1 → 0.0 (unknown excluded rather than guessed wrong).
- **Code — PlanetRepository.kt**: `effectiveUsed = bufferUsed` → `effectiveUsed = if (dedicatedCapacity > 0) dedicatedUsed else bufferUsed`. Planets with Launchpad/SF now show those structures' fill (export readiness). Planets with only routing hubs show hub buffer fill.
- **Root causes**: Storm (40132056) showed 100%/11k — type 2308 missing from P0 list caused 20× volume inflation. Barren Launchpad showed 25% — hub contents were counted against LP capacity. Gas buffer was 2× actual — P0 volume was 0.01 instead of 0.005.
- Wiki: [[Planet]] — business rules updated (pending).

## [2026-07-13] dev | Wallet journal — description field + filter + 5 entries
- **Code — EsiWalletJournalEntryDto.kt**: added `description: String = ""` field.
- **Code — WalletJournalEntry.kt**: added `description: String = ""`; improved `displayName` mapping: `market_transaction` now shows "Market sale" / "Market buy" based on sign; added `planetary_construction` → "PI construction", `transaction_tax` → "Sales tax", `air_career_program_reward` → "AIR reward".
- **Code — CharacterRepository.kt**: `observeWalletJournal` now filters `market_escrow` entries (escrow releases = mechanical noise, not real transactions) and takes 5 instead of 3.
- **Code — DashboardScreen.kt**: `WalletJournalRow` uses `entry.description.ifEmpty { entry.displayName }` as primary label; added `TextOverflow.Ellipsis` (maxLines=1).
- **Trigger**: user's wallet journal export showed top-3 as all `planetary_construction` (PI colony build costs from Jul 11), hiding the 901M ISK market sale from Jul 10. Filtering escrow + 5 entries surfaces meaningful activity.
- Wiki: [[WalletJournalEntry]] created; [[Screen - Dashboard]] implementation notes updated.

## [2026-07-13] dev | Wallet journal — also filter planetary_construction
- **Code — CharacterRepository.kt**: added `planetary_construction` to `JOURNAL_NOISE_TYPES`. A 5-planet EVE colony setup produces 45+ `planetary_construction` journal entries (structure build costs ~32.7M ISK total), which dominated Recent Activity even after the earlier `market_escrow` filter. The 901M ISK market sale on Jul 10 was still invisible.
- **Filter logic**: `JOURNAL_NOISE_TYPES = setOf("market_escrow", "planetary_construction")`. Top 5 after filtering: market_transaction +901M, transaction_tax -45M, two air_career_program_reward, skill_purchase -1.3M.
- Wiki: [[WalletJournalEntry]] business rules updated; [[Screen - Dashboard]] implementation notes updated.

## [2026-07-13] dev | PI storage display — fix type IDs from ESI data
- **Root cause**: `PiVolumeTable.launchpadTypeIds` was entirely wrong — included Barren CC (2524), Gas CC (2534), and fabricated IDs. `STORAGE_FACILITY_TYPE_ID = 2257` was also wrong; SFs have planet-type-specific IDs. Result: Barren showed "Launchpad 0%" (CC pin had no contents but was counted, doubling capacity denominator).
- **Code — PiVolumeTable.kt**: replaced `launchpadTypeIds` with ESI-verified set {2544 Barren, 2555 Lava, 2557 Storm, 2543 Gas, 2256 Temperate}; added `storageFacilityTypeIds` {2541 Barren, 2558 Lava, 2561 Storm, 2536 Gas, 2562 Temperate} at 12,000 m³; removed wrong `STORAGE_FACILITY_TYPE_ID = 2257`; updated `capacityOf()` and `storageTypeIds`.
- **Code — PlanetRepository.kt**: removed debug `println` statements; updated comment (2524 is CC, not LP).
- **Wiki**: [[Planet]] entity page gained full type ID taxonomy table (CC, Extractor, BIF, AIF, SF, LP per planet type), all verified from live ESI colony responses.

## [2026-07-13] dev | PI screen — pull-to-refresh
- **Code — PlanetViewModel.kt**: added `_isRefreshing: MutableStateFlow<Boolean>`, exposed as `isRefreshing: StateFlow<Boolean>`; extracted `loadPlanets()` private function that attaches `.onCompletion { cause -> if (cause == null) _isRefreshing.value = false }` so the spinner clears after the ESI fetch finishes (handles both success and error-with-cache cases correctly); `refresh()` now sets `_isRefreshing.value = true` before bumping `refreshTrigger`.
- **Code — PiScreen.kt**: added `PullToRefreshBox` wrapping `PiSuccess` content; `PiContent` and `PiSuccess` receive `isRefreshing: Boolean` + `onRefresh: () -> Unit`; file-level `@OptIn(ExperimentalMaterial3Api::class)` added.
- **Behaviour**: PTR spinner appears on pull, stays while ESI fetch is in flight (SWR flow: stale cache emits first, spinner stays, fresh ESI data arrives → spinner stops). On ESI error with cache, spinner also stops (`onCompletion` fires either way when flow block ends normally).

## [2026-07-13] dev | PI — split Storage Facility and Launchpad display
- **Code — ColonySummary.kt**: replaced single `storageFillRatio/storageCapacityM3/storageUsedM3/storageLabel/hasPassiveStorage` with separate `sfCapacityM3/sfUsedM3` (Storage Facility, 12,000 m³ P0 buffer) and `lpCapacityM3/lpUsedM3` (Launchpad, 10,000 m³ finished goods). `sfFillRatio`/`lpFillRatio` computed as derived properties.
- **Code — PlanetRepository.kt**: `toColonySummary()` simplified — removed allPassivePins/bufferUsed/effectiveCapacity fallback logic; now filters SF and LP pins by `PiVolumeTable.storageFacilityTypeIds` / `launchpadTypeIds` directly and sums their contents. Added import for `EsiColonyPinDto`.
- **Code — PiScreen.kt**: `StorageRow(colony)` replaced with parameterised `StorageRow(label, fillRatio, usedM3, capacityM3)`; called twice per planet card when capacity > 0 — "Storage" row first, "Launchpad" row second. Previews updated with real Nakugard ESI values.
- **Why separate**: SF fill signals P0 buffer state (extractors filling up → check routes); LP fill signals P2/P3 readiness for collection. Combined ratio was meaningless (7% of 22,000 m³).

## [2026-07-13] dev | PI — SecureStorage crash fix + Planet.md type ID correction
- **Bug**: `AEADBadTagException` crash on startup after app reinstall — Android Keystore invalidates the encryption key; `EncryptedSharedPreferences` fails to decrypt old keyset.
- **Code — SecureStorage.android.kt**: wrapped `EncryptedSharedPreferences.create()` in try/catch; on exception, deletes stale `eve_secure_prefs` file via `context.deleteSharedPreferences()` and recreates. User must re-authenticate after reinstall.
- **Wiki fix — [[Planet]]**: type 2481 was incorrectly classified as Temperate AIF. ESI data for planet 40132079 (Nakugard VI, Temperate) shows both type-2481 pins use schematics 131 and 134 (P0→P1 conversion = BIF behaviour). Moved 2481 to BIF table; Temperate AIF type ID now TBD (no AIF observed on Temperate colony yet).
- **Confirmed from live ESI**: Storm CC=2550, Storm ECU=3067, Storm BIF=2483, Storm AIF=2484 (planet 40132056). Temperate CC=2254, Temperate ECU=3068, Temperate BIF=2481 (planet 40132079). Nakugard VI genuinely has 2 SF + 2 LP (P0→P1 production colony); 24,000/20,000 m³ capacity is correct in-game setup.

## [2026-07-14] dev | PI data freshness + iOS warning fixes
- **Code — ColonySummary.kt**: added `dataFetchedAtEpochSeconds: Long` field; added `dataAgeText(nowEpochSeconds: Long): String` → "just now" / "Xm ago" / "Xh ago".
- **Code — PlanetRepository.kt**: captured `fetchedAt = Clock.System.now().epochSeconds` right after `esiApi.fetchColony()` returns; passed to `toColonySummary(fetchedAt)`.
- **Code — PiScreen.kt**: added "data Xm ago" label (10 sp, 30 % alpha) per colony section via `colony.dataAgeText(now)`.
- **Code — App.kt, DashboardScreen.kt, JobsScreen.kt, PiScreen.kt, SkillsScreen.kt**: `@Preview` import corrected from `org.jetbrains.compose.ui.tooling.preview.Preview` → `androidx.compose.ui.tooling.preview.Preview`.
- **Code — shared/build.gradle.kts**: added `-Xexpect-actual-classes` to `freeCompilerArgs` to suppress beta warning for `expect`/`actual` class declarations.
- **Code — SecureStorage.ios.kt**: added `@OptIn(kotlinx.cinterop.BetaInteropApi::class)` for `readBytes()` extension.
- **Code — Crypto.ios.kt**: removed redundant `.toInt()` on `CC_SHA256_DIGEST_LENGTH`.
- **Why freshness**: ESI colony endpoint caches ~10 min. User saw extractors showing 22 h remaining in app while EVE client showed 2 d 23 h — ESI was serving cached data. Label tells user when to force-refresh.
- Wiki: [[Screen - PI]] implementation notes + current UI version table updated.

## [2026-07-14] dev | PI screen — text legibility fix (font size + brightness)
- **Bug**: PI screen text diverged from Dashboard/Jobs/Skills — those three always use `MaterialTheme.typography.*` tokens (floor: `labelSmall` 11sp) with `onSurfaceVariant` at full opacity. `PiScreen.kt` instead hardcoded raw `fontSize` down to 9–10sp and stacked extra `.copy(alpha = 0.3f–0.6f)` on top of the already-muted `onSurfaceVariant` — worst offender was the "data Xm ago" label at 10sp / 0.3 alpha, effectively invisible.
- **Code — PiScreen.kt**: `ExtractorCountdown` "STOPS IN"/"EXTRACTORS" label 9sp→11sp, alpha 0.5→full. `FactoriesRow`/`StorageRow` labels ("Factories"/"Storage"/"Launchpad") gained `FontWeight.Medium`, alpha 0.6→full. Factory count, m³ text, "data Xm ago" alpha 0.4/0.3→full ("data Xm ago" also 10sp→11sp). Icon tints bumped 0.4→0.6 (kept muted deliberately — decorative, not information).
- **User-facing trigger**: user reported PI tab text was "почти не читаем" (barely readable) compared to other tabs; confirmed via side-by-side comparison of exact fontSize/alpha values across all 4 screens.
- Wiki: [[Screen - PI]] implementation notes + current UI version table updated with the new text-floor convention (11sp / full `onSurfaceVariant`, icons excepted).

## [2026-07-14] lint | Ktlint + Detekt sweep after PI legibility fix
- **Trigger**: user asked whether we check code after writing it — answer was no, ktlint/detekt hadn't been run since the crash-fix + freshness-label commit. Ran them; found real issues.
- **Ktlint**: 39 formatting violations across `EsiColonyDto.kt`, `PlanetEsiApi.kt`, `CharacterRepository.kt`, `PlanetRepository.kt`, `PiVolumeTable.kt` (all touched earlier this session, never formatted). Fixed via `ktlintFormat`.
- **Detekt — SecureStorage.android.kt**: `catch (e: Exception)` was `TooGenericExceptionCaught` + `SwallowedException`. Narrowed to `catch (e: GeneralSecurityException)` / `catch (e: IOException)` (the two types `EncryptedSharedPreferences.create()` actually declares); kept `@Suppress("SwallowedException")` on the property with the existing comment as justification — any crypto/IO failure on the stale keystore file should trigger the same wipe-and-recreate, so the swallow is intentional.
- **Detekt — PiScreen.kt `StorageRow`**: `MultipleEmitters` (5 top-level composable siblings, no wrapping container) — wrapped the body in `Column { }`. No visual change (Column has no arrangement spec, same stacking as before).
- **Detekt — PiScreen.kt `PiPreviewSuccess`**: `LongMethod` (82 vs 80 lines) after ktlintFormat expanded `Planet(...)` positional args one-per-line. Added `@Suppress("LongMethod")` — pure preview fixture data, consistent with existing baselined preview noise in this file.
- Regenerated `composeApp/detekt-baseline.xml` (annotation change shifted the ID for the already-baselined `UnusedPrivateMember` on `PiPreviewSuccess`).
- **Verified end-to-end**: `ktlintCheck` + `detekt` clean on both modules, `compileDebugKotlinAndroid` clean, then built and installed the debug APK on the connected device (`58090DLCQ009DN`) and screenshotted the live PI screen — confirmed "Factories", "Storage"/"Launchpad" labels, m³ values, and "data just now" are all legible against a different (non-Ember) active theme, confirming the fix operates correctly through the `onSurfaceVariant` token rather than a hardcoded color.

## [2026-07-14] dev | Completion notifications — skill queue, industry jobs, PI extractors
- **Trigger**: user asked to audit the ADRs for notification-related design notes, check what's tracked, and build it. Audit found [[ADR-006 - Android Foreground Service]] / [[ADR-007 - iOS Local Notifications]] speced skill-queue notifications that were never implemented (manifest `<service>` commented out, zero notification code anywhere); industry jobs and PI extractors had no notification design at all.
- **Shared**: new `NotificationScheduler` expect/actual (`shared/.../platform/NotificationScheduler.kt`) + `NotificationSource` enum (SKILL/INDUSTRY_JOB/PI_EXTRACTOR) + `ScheduledCompletion` data class. `reconcile(source, items)` cancels everything previously scheduled for that source and reschedules fresh — no incremental diffing needed at this scale.
- **Repositories**: `SkillQueueRepository`, `IndustryJobRepository`, `PlanetRepository` each gained a `NotificationScheduler` constructor param and call `reconcile()` right after their existing ESI fetch, using the freshly-fetched in-memory values — no new ESI calls, no DB migration (in particular, `Planet`'s extractor expiry is never persisted to SQLDelight; scheduling reads it straight off the in-memory colony snapshot).
- **Android**: `SkillTrainingService` (new, `shared/androidMain/.../platform/service/`) — ForegroundService with a per-second-ticking live-countdown notification (finishes the ADR-006 ambition), backed by an `AlarmManager` exact-alarm fallback for when the OS kills the service. `NotificationAlarmReceiver` posts one-shot notifications for industry jobs and PI extractors (plain `AlarmManager`, no live countdown — multiple simultaneous jobs/planets would make a per-instance ticker clutter). Two corrections vs. ADR-006's original wording: `foregroundServiceType="specialUse"` not `"dataSync"` (targetSdk 35 caps `dataSync` FGS at 6h/24h — fatal for multi-day skill training), and `SCHEDULE_EXACT_ALARM` not `USE_EXACT_ALARM` (Play policy restricts the latter to core alarm/clock apps). Android has no API to enumerate pending alarms, so `NotificationScheduler.android.kt` keeps its own per-source `SharedPreferences` bookkeeping of scheduled ids.
- **iOS**: `NotificationScheduler.ios.kt` — all three sources (including skill queue) use one-shot `UNNotificationRequest` via `UNTimeIntervalNotificationTrigger` (no live-countdown analog exists on iOS, unchanged from ADR-007).
- **Permission request**: new `RequestNotificationPermissionEffect()` `@Composable expect`/`actual` (mirrors the existing `rememberHapticFeedback()` pattern) — lives in `composeApp`, not part of `NotificationScheduler`, since Android's runtime `POST_NOTIFICATIONS` request needs an Activity-bound launcher the shared-module actual can't obtain. Called once from `App.kt`'s `MainApp()`.
- **Gradle**: added `androidx-core-ktx` to `shared`'s `androidMain` deps (NotificationCompat/ServiceCompat), `compose-activity` to `composeApp`'s `androidMain` deps (permission launcher).
- **Manifest**: replaced `FOREGROUND_SERVICE_DATA_SYNC`/`USE_EXACT_ALARM` with `FOREGROUND_SERVICE_SPECIAL_USE`/`SCHEDULE_EXACT_ALARM`; uncommented and finalized the `SkillTrainingService` `<service>` block (fully-qualified name — the class lives in `shared`, not `androidApp`) with its `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` justification; added the `NotificationAlarmReceiver` `<receiver>` block.
- **New resource**: `shared/src/androidMain/res/drawable/ic_notification.xml` (`shared/androidMain` had no `res/` directory at all before this — confirmed AGP picks it up automatically, no extra Gradle wiring needed).
- Caught and fixed two of my own mistakes during iOS compilation: `UNMutableNotificationContent.title/body/sound` are read-only `val` on the Kotlin/Native binding (compiler: `'val' cannot be reassigned`) — the ObjC covariant-property-redeclaration doesn't become a Kotlin `var` override, so the setters are separate `setTitle()`/`setBody()`/`setSound()` calls, not property assignment. Also had to `filterIsInstance<UNNotificationRequest>()` on the `getPendingNotificationRequestsWithCompletionHandler` callback's list — its element type didn't infer to `UNNotificationRequest` on its own.
- **Verified**: `ktlintCheck` + `detekt` clean on both modules; `compileDebugKotlinAndroid` + `compileKotlinIosSimulatorArm64` clean; `:androidApp:assembleDebug` succeeds (manifest merge + new drawable resource link cleanly). **Not yet verified**: live on-device behavior (permission prompt, live countdown ticking, alarms actually firing) — no Android device or emulator was connected this session.
- Wiki: [[ADR-015 - Unified Completion Notifications]] created; [[Skill Queue]], [[Industry Job]], [[Planet]] lifecycle sections updated; [[ADR-006 - Android Foreground Service]] and [[ADR-007 - iOS Local Notifications]] References sections gained forward-links (not superseded — their core decisions still stand).

## [2026-07-15] dev | Completion notifications — device verification, 3 real bugs found + fixed
- **Trigger**: continuation of the 2026-07-14 notification work — no device was connected then, so live behavior was unverified. User connected an Android device and logged in for real; this entry covers driving the actual app end-to-end and what it found.
- **Bug 1 — duplicate skill notification**: `SkillTrainingService`'s `startForeground()` anchor post (id=1001, tag=null) and the per-second tick loop's `notify()` (tag="skill_training_live", id=0) used different notification keys — confirmed via `dumpsys notification`/shade screenshot showing two permanent, non-dismissible "Coherent Ore Processing → Level 4" entries with different countdown values instead of one updating notification. **Fix**: unified every SKILL-source post (anchor, tick, backup-alarm completion) onto one fixed `Int` id, `SKILL_LIVE_NOTIFICATION_ID = 1001` (`ServiceCompat.startForeground()` only accepts a plain id, which is what made the string-tag scheme unsound). Confirmed fixed: `dumpsys notification` now shows exactly one `id=1001, tag=null` record; shade shows one live-ticking "Interplanetary Consolidation → Level 5 / 544h 30m 45s remaining" entry.
- **Bug 2 — missing-channel crash**: probing the fix by starting `SkillTrainingService` directly (bypassing the normal repository → `NotificationScheduler` → Service chain, since re-authenticating myself with the user's EVE credentials isn't something I do) crashed with `FATAL EXCEPTION: CannotPostForegroundServiceNotificationException: Bad notification for startForeground` / `No Channel found for pkg=..., channelId=skill_training`. The Service assumed `NotificationScheduler`'s constructor had already created its channel. **Fix**: extracted idempotent `ensureNotificationChannels()`, called from both `NotificationScheduler.init{}` and `SkillTrainingService.onCreate()`. Confirmed fixed: identical repro steps, no more FATAL EXCEPTION.
- **Bug 3 — wrong skill picked (real, not a test artifact)**: added temporary `Log.d`/`println` tracing (kept the `NotificationScheduler.android.kt` trace-level logging permanently; removed the repository-level dump once done) and found `SkillQueueRepository`'s `head = freshEntries.firstOrNull { it.queuePosition == 0 && it.isTraining }` was picking **"Coherent Ore Processing"** (already finished ~6.4h earlier) instead of **"Interplanetary Consolidation"** (the real, currently-training skill, sitting at `queuePosition == 1`) — because ESI hadn't rotated the finished entry out of `queuePosition == 0` yet. This is the exact quirk already documented on [[Skill Queue]] ("Completed entries remain in the response"), which `DashboardViewModel`'s `activeSkill` selection already handles correctly (`firstOrNull { it.isTraining && !it.hasFinished(now) }`) but my new notification-scheduling code didn't. **Fix**: matched the same selector. Confirmed fixed: logs show the correct skill + a 22.68-day-future timestamp matching the UI exactly.
- **Also confirmed working, no bugs**: PI extractor scheduling (`reconcile(PI_EXTRACTOR, 5 items, 3 future)` — 2 of 5 planets' extractors already stopped, correctly excluded; matches "STOPS IN 1d 3h" ×2 and "stopped" ×1 visible on screen) and industry job scheduling (`reconcile(INDUSTRY_JOB, 1 items, 0 future)` — the one active job is already "Ready to deliver" in the UI, correctly excluded). `dumpsys alarm` showed exactly 4 scheduled alarms (3 extractor + 1 skill backup) matching expectations. Zero `FATAL EXCEPTION`s in the final clean pass.
- **Also learned**: `pm clear` does not cancel an app's `AlarmManager` alarms (system-service-owned, not app-data — only a full uninstall does); an APK **update** (not just fresh install) revokes the `SCHEDULE_EXACT_ALARM` grant (`AlarmManager: ... lost permission to set exact alarms!`), confirming the inexact-delivery fallback path is real and regularly hit, not theoretical.
- **Verified**: `ktlintCheck` + `detekt` clean on both modules; `compileDebugKotlinAndroid` + `compileKotlinIosSimulatorArm64` clean after all fixes. **Not verified this pass**: iOS device/simulator behavior.
- Wiki: [[ADR-015 - Unified Completion Notifications]] updated with all 3 findings + the exact-alarm-revocation note; [[Skill Queue]] business rules sharpened to call out that the completed-entry quirk applies to any "pick the active skill" logic, not just rendering; `index.md` open thread resolved.

## [2026-07-15] dev | selfcheck skill created + notification polish (day-format, capsule icon)
- **selfcheck skill**: new project skill at `.claude/skills/selfcheck/SKILL.md` — user asked to split "verify" (already an existing bundled skill; drives the real app) from a separate static pass over uncommitted work. `selfcheck` always runs `ktlintCheck`/`detekt` first, then re-reads the diff as a skeptical second pass specifically hunting for the bug classes found during today's earlier verification (mismatched selectors for the same domain concept across files, inconsistent identifiers/keys across call sites, exception-handling boundary leaks, contradictions with vault-documented ESI quirks).
- **Notification day-format fix**: `SkillTrainingService`'s ticking notification used `formatHms()` (raw hours, e.g. "544h 30m 45s remaining" for a 22-day skill) instead of `formatDhm()` (day-rollover, "22d 15h 55m") — the same helper `ActiveSkillProgressSection` already uses for the Dashboard/Skills hero countdown. Since `formatDhm()` only changes once a minute, also slowed the tick loop from 1s to 60s — sub-minute ticks were pure waste (identical text) and had already tripped Android's notification-rate limiter once during verification.
- **Capsule notification icon**: replaced the placeholder Material bell (`ic_notification.xml`) with the actual traced EVE capsule silhouette already living in `PodSplashScreen.kt`'s `buildBlobPath()` (188 cubic curves, traced from a real capsule via Vectorizer.io) — extracted programmatically (small Python script parsing the Compose `Path` calls and re-emitting them as Android vector `pathData` scaled to a 24×24 viewport) rather than hand-transcribed, to avoid transcription errors across ~190 curve commands. Confirmed rendering correctly in the status bar on-device (zoomed crop of a screenshot) — the notification list row itself doesn't show a small-icon badge on this launcher, only the status bar does, which tripped up verification briefly.
- Both changes verified on-device (Pixel 10 Pro XL): shade shows "22d 15h 55m remaining"; status bar shows the pod silhouette instead of a bell. `ktlintCheck`/`detekt`/Android+iOS compile all clean.

## [2026-07-15] dev | Unit test scope, SkillsScreen consolidation, AppContext removal, Android Lint sweep
- **Trigger**: follow-up work after the notification feature landed — user asked to consult on unit vs. UI test scope (decided to start with unit tests only, Compose snapshot/UI tests explicitly descoped for now — no mature KMP-wide screenshot-testing story exists across Android+iOS in one suite), consolidate `SkillsScreen.kt` onto the shared `activeSkill()` selector, investigate lint warnings surfacing in `SkillTrainingService.kt`, and replace the `AppContext` static holder with idiomatic context passing.
- **Unit tests**: 7 new `commonTest` files (`SkillQueueEntryTest`, `PlanetTest`, `ColonySummaryTest`, `IndustryJobTest`, `WalletJournalEntryTest`, `PiVolumeTableTest`, `EsiErrorMapperTest`) covering domain-model business rules, the PI extractor volume table, and `EsiErrorMapper`'s user-facing message mapping. Uses the existing injectable-`Clock` pattern from `SkillProgressCalculatorTest` for deterministic time-based cases. All pass on both `shared:testDebugUnitTest` (JVM) and `shared:iosSimulatorArm64Test` (Kotlin/Native).
- **SkillsScreen consolidation**: `SkillsScreen.kt` had its own inline copy of the "pick the training skill that hasn't finished yet" logic, independent from `DashboardViewModel` and `SkillQueueRepository` (the site of the [[Skill Queue]] completed-entry-quirk bug found during notification device-verification). Extracted `List<SkillQueueEntry>.activeSkill(nowEpochSeconds)` onto `SkillQueueEntry.kt` and pointed all three call sites at it, so the quirk only needs handling correctly in one place going forward.
- **AppContext removal**: see [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]] (Consequences addendum) for the convention this now follows. `AppContext.kt` deleted; `SecureStorage`, `DatabaseDriverFactory`, `NotificationScheduler` (Android actuals) take `Context` via constructor, wired through the pre-existing `androidContext()` Koin registration.
- **Android Lint sweep** (a third, distinct tool from ktlint/detekt — first run this session): `MissingPermission` on `POST_NOTIFICATIONS`-gated `notify()` calls in `SkillTrainingService`/`NotificationAlarmReceiver`/`NotificationScheduler.scheduleAlarm` — empirically, lint's detector doesn't recognize a permission check delegated through a named helper function, and is sensitive to whether the checked receiver is `this` vs. a `Context` parameter (the latter, in `NotificationAlarmReceiver`, never got recognized even inlined — suppressed with a comment). Also suppressed `InlinedApi` (`FOREGROUND_SERVICE_TYPE_SPECIAL_USE` referenced below its API 34 floor is intentional and safe — see [[ADR-015 - Unified Completion Notifications]]) and `VectorPath` (the capsule icon's full-fidelity 188-curve trace is deliberate, not an oversight).
- **Verified**: `ktlintCheck`/`detekt`/Android Lint (`lintDebug`) clean across all 3 modules (`shared`, `composeApp`, `androidApp`); unit tests pass on both KMP targets; `androidApp:assembleDebug` packages successfully.
- Committed as 4 separate commits (tests / consolidation / AppContext removal / lint sweep) rather than one bundled commit, at the user's request.
- Wiki: [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]] Consequences updated with the context-passing convention; [[Skill Queue]] unaffected (the fix reuses its already-documented quirk, no new business rule).

## [2026-07-15] dev | Market-launch readiness review + Guide - App Store Launch Readiness
- **Trigger**: user asked for a review of what's needed before a real Play Store / App Store launch — what exists today, what a companion app like this generally needs, a prioritized plan, and updated specs.
- **Audit** (vault + codebase + external research, no code changes this pass): confirmed the app's core feature set is done and committed, but almost everything *outside* it is missing — no Android release signing, no iOS display name, no CCP/EVE disclaimer, no privacy policy, no Play Data Safety/Apple nutrition-label prep, no crash reporting, no CI/CD, no ESI `User-Agent` header, no ESI rate-limit (420) handling, no ESI cache-header (`Expires`/`ETag`) compliance, no `CharacterOwnerHash` usage (raw `characterId` used everywhere), no offline-detection. Also surfaced: Google Play requires targetSdk 36 by **2026-08-31** (app is on 35) and Apple's Privacy Manifest (`PrivacyInfo.xcprivacy`) is likely required given SQLDelight/Kotlin-Native file-access APIs.
- **New guide**: [[Guide - App Store Launch Readiness]] — the master P0 (submission blockers) / P1 (strongly recommended) / P2 (backlog) tracker, with per-item technical notes, file references, and explicit "requires your own action" flags for anything needing an account, credential, or hosting/wording choice only the user can make (release keystore, crash-reporter account, Play Console/App Store Connect forms, privacy-policy hosting).
- **Vault hygiene found + fixed along the way**: [[ADR-013 - Faction Color Themes]] and [[ADR-014 - Ktlint Detekt Linter Setup]] recovered as proper ADR files — `index.md` referenced both since 2026-07-10, but neither file ever existed; reconstructed from their `log.md` entries. Also found and fixed a stale OAuth redirect-URI documented as `eve-tracker://callback` across [[ADR-008 - OAuth2 PKCE via System Browser]] (Accepted, corrected via addendum), [[Platform - Android]], [[Platform - iOS]], and [[OAuth2 PKCE]] — the actually-shipped value has been `eveauth-podforeve://callback` since 2026-07-08.
- [[ADR-010 - Platform Targets]] gained an addendum flagging the Aug 31 2026 targetSdk-36 deadline against its existing "bump annually" policy.
- No app code changed in this pass — everything above is vault documentation. Implementation of roadmap items happens turn-by-turn, tracked via the new guide's Status column.
- Wiki: [[Guide - App Store Launch Readiness]] created; [[ADR-013 - Faction Color Themes]], [[ADR-014 - Ktlint Detekt Linter Setup]] recovered; [[ADR-008 - OAuth2 PKCE via System Browser]], [[ADR-010 - Platform Targets]], [[Platform - Android]], [[Platform - iOS]], [[OAuth2 PKCE]] updated; `index.md` Guides section + Open threads updated.

## [2026-07-16] dev | GitHub Actions CI — lint, static analysis, tests, debug build
- **Trigger**: user asked to set up CI on GitHub Actions — build, tests, lint all considered essential before release/publishing.
- **New**: `podForEveOnline/.github/workflows/ci.yml`, two jobs, both triggered on push/PR to `master` plus manual `workflow_dispatch`:
  - `lint-and-android` (ubuntu-latest): `ktlintCheck detekt shared:lintDebug composeApp:lintDebug androidApp:lintDebug shared:testDebugUnitTest androidApp:assembleDebug` — uploads the debug APK as a build artifact on success, uploads lint/test reports on failure.
  - `ios-test` (macos-latest, needed for the Kotlin/Native Xcode toolchain): `shared:iosSimulatorArm64Test`.
- **No secrets required**: confirmed by reading the actual Gradle wiring — `shared/build.gradle.kts:64` defaults `ESI_CLIENT_ID` to `""` when `local.properties` is absent (`.takeIf { it.exists() }`), and iOS's `EsiClientId.ios.kt` reads from the *app's* `Info.plist` at Xcode-build time, which `shared`'s Kotlin/Native compilation never touches. Both only affect real SSO login at runtime, not compilation or unit tests. This corrects [[ADR-011 - Secrets via expect-actual and local.properties]]'s original (broader-than-necessary) Consequences claim that CI needs secret injection — addendum added there.
- **Verified before committing**: ran the exact command sequence from both jobs locally (`BUILD SUCCESSFUL` for both), and confirmed the debug-APK output path (`androidApp/build/outputs/apk/debug/*.apk`) matches the artifact-upload step.
- **Still open**: a signed release-build job is a deliberate follow-up, gated on [[Guide - App Store Launch Readiness]]'s P0 keystore item — that's the point secrets actually become necessary in CI.
- Wiki: [[Guide - App Store Launch Readiness]] P1 #7 marked done with details; [[ADR-011 - Secrets via expect-actual and local.properties]] addendum correcting the CI-secrets Consequences claim.

## [2026-07-16] dev | P0 launch-readiness items: iOS display name, CCP disclaimer, privacy policy draft, targetSdk 36, Apple Privacy Manifest
- **Trigger**: user asked to tackle the remaining P0 items from [[Guide - App Store Launch Readiness]] (everything except release signing, which is blocked on the user checking their Apple Developer account status) while a separate Fastlane/iOS-signing conversation was parked.
- **iOS app display name**: `INFOPLIST_KEY_CFBundleDisplayName = PodForEve` added to both Debug/Release configs in `iosApp.xcodeproj/project.pbxproj`. Verified via `xcodebuild -showBuildSettings` (not just eyeballing the diff) that it resolves correctly.
- **CCP/EVE disclaimer**: added to `LoginScreen.kt` below the existing "Your EVE Online companion" copy — standard fan-tool wording confirmed during the original audit.
- **Privacy policy**: drafted (not published) at `podForEveOnline/PRIVACY_POLICY.md`. Checked `AuthRepository.logout()` first — confirmed it only clears the refresh token, not the local SQLDelight cache — so the policy honestly states cached data persists until uninstall, with an inline TODO tying this to [[Guide - App Store Launch Readiness]] P1 #5. Cited CCP's authorized-apps revocation page — verified the URL is real (redirects to CCP's actual SSO login domain) before including it in a legal document.
- **Android targetSdk 36**: bumped in `gradle/libs.versions.toml` (`compileSdk` was already 36). Full lint/detekt/test/build verification, then **real device-verify** on the connected Pixel 10 Pro XL (per this project's established rigor — compiling isn't enough for a targetSdk bump, since Android version behavior changes are often silent at compile time): installed, navigated all 4 tabs, opened/closed the theme-switcher bottom sheet. Found back-button-from-a-tab exits straight to the launcher instead of returning to Dashboard — **A/B tested against a targetSdk-35 rebuild of the identical code and confirmed the same behavior there**, so this is pre-existing (a likely Voyager `TabNavigator` characteristic), not a targetSdk-36 regression. Logged as a new P2 item rather than fixed, since it's out of scope for this pass.
- **Apple Privacy Manifest**: added JetBrains' `apple-privacy-manifests` Gradle plugin (1.0.0) to `shared`, authored `shared/PrivacyInfo.xcprivacy` declaring the two categories Compose Multiplatform's iOS runtime is documented to need (FileTimestamp reason `0A2A.1`, SystemBootTime reason `35F9.1` — verified against kotlinlang.org's own docs, not guessed). Attempted JetBrains' deeper `required_reason_finder.py` symbol scanner to check other dependencies for additional categories; it failed on a Gradle configuration-cache incompatibility with this project's Gradle 9.4.1 (script targets an older baseline) — read the whole script before running it (legitimate: downloads a pinned Kotlin/Native distribution to `~/.konan` for `klib dump-ir-signatures`, temporarily patches and restores `build.gradle.kts`, no destructive or suspicious behavior), then cleaned up the 854MB diagnostic-only K/N distribution it left behind since it was orphaned dead weight unrelated to the project's real toolchain. Verified the two declared categories are sufficient for now by rebuilding the iOS simulator framework and confirming `PrivacyInfo.xcprivacy` embeds byte-identical inside `shared.framework`.
- **Verified end-to-end**: `ktlintCheck`/`detekt`/Android Lint (3 modules)/`shared:testDebugUnitTest`/`shared:iosSimulatorArm64Test`/`androidApp:assembleDebug` all clean at the final state (targetSdk 36, privacy manifest plugin wired, disclaimer added).
- Wiki: [[Guide - App Store Launch Readiness]] P0 items 2/3/4/6/7 marked done (4 marked drafted-not-published); new P2 item added for the back-navigation finding.

## [2026-07-17] dev | AGP KMP-library plugin migration — shared + composeApp
- **Trigger**: user asked to migrate off the deprecated `com.android.library` + KMP plugin pairing "while the codebase hasn't grown even more" — this bypass was already flagged as deferred debt in [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]]. Investigation revealed real added scope before starting: the new plugin doesn't support `BuildConfig` at all (breaks [[ADR-011 - Secrets via expect-actual and local.properties]]'s exact secret-injection mechanism), and ADR-012 itself already documented a real risk — `android.newDsl=false` was needed for Voyager's `libraryVariants` usage to work. User confirmed doing the full migration anyway, treating the Voyager risk as the first gate to clear rather than something to discover late.
- **Migrated**: `shared` and `composeApp` → `com.android.kotlin.multiplatform.library`; `androidApp` dropped the redundant `org.jetbrains.kotlin.android` plugin (AGP 9 built-in Kotlin); `gradle.properties`'s `android.builtInKotlin`/`android.newDsl` flipped to `true` (unavoidably atomic across all 3 modules — confirmed no partial-migration state is possible).
- **Secret injection redesign**: replaced `buildConfigField`/`BuildConfig.ESI_CLIENT_ID` with a custom `GenerateEsiConfigTask` generating a plain Kotlin file — chosen over the community BuildKonfig plugin (see [[ADR-016 - AGP KMP Library Plugin Migration]] for the full rationale: matches this project's existing dependency-light convention, avoids BuildKonfig's non-standard generated-output path colliding with the ktlint exclude filter).
- **Two real syntax errors hit and fixed**, both from following the official migration doc's exact-but-non-working sample: `compilerOptions.configure { jvmTarget.set(...) }` inside `kotlin { android { } }` doesn't resolve (plain `compilerOptions { }` does); `androidMain.kotlin.srcDir(...)` as a standalone statement doesn't resolve (needs to be inside an `androidMain { dependencies {...}; kotlin.srcDir(...) }` block).
- **The Voyager gate, cleared**: got to a compiling `androidApp:assembleDebug` first and device-verified on the connected Pixel 10 Pro XL *before* any further work — existing login session persisted, live ESI data fetched correctly (ISK/SP/skill queue/PI/jobs all populated), all 4 bottom-nav tabs navigated cleanly, and the theme-switcher bottom sheet opened/closed correctly. ADR-012's documented `libraryVariants` risk did not reproduce.
- **Pre-existing gap surfaced, fixed as a necessary side-effect**: adopting `shared:check` (instead of hand-named old task strings) revealed SQLDelight's `verifyMigrations = true` had never actually been exercised — no baseline schema file existed for it to check against, unrelated to this migration, just never triggered by any command this project had run before. Fixed by running `:shared:generateCommonMainAppDatabaseSchema` once (safe, additive, no behavior change) to produce the missing `1.db` baseline.
- **Known new gap, documented not fixed**: Android Lint's `check` integration on `shared`/`composeApp` — the new plugin exposes `lintAnalyzeAndroidHostTest` but nothing equivalent to the old `lintDebug` wired into `check`. `androidApp`'s Android Lint is unaffected. Flagged in ADR-016 as a watch item.
- **Verified end-to-end**: `shared:check composeApp:check androidApp:lintDebug shared:iosSimulatorArm64Test shared:compileKotlinIosSimulatorArm64 androidApp:assembleDebug` all clean, confirmed via actual test-result XML counts (zero failures/errors, both Android-host and iOS Kotlin/Native suites) — not just "BUILD SUCCESSFUL". Updated `.github/workflows/ci.yml`'s task names to match (`lintDebug`/`testDebugUnitTest` → `check`) and re-verified that exact new command sequence locally too.
- Wiki: [[ADR-016 - AGP KMP Library Plugin Migration]] created; [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]] addendum noting the deferred migration landed; [[Guide - App Store Launch Readiness]] P1 #7 updated with the new task names; `index.md` updated.

## [2026-07-17] dev | Dependency audit + androidx/Chucker/compileSdk 37 bumps
- **Trigger**: user asked to check what library updates are available now that the AGP migration ([[ADR-016 - AGP KMP Library Plugin Migration]]) is done, with an explicit preference: stable versions only, alpha acceptable solely when no stable option exists.
- **Web research hit real reliability limits**: several sources (Koin, detekt, kotlinx-coroutines, kotlinx-serialization, SQLDelight) gave mutually contradictory "latest version" numbers, some *older* than what's already pinned — flagged as unreliable rather than acted on. Google's own Maven metadata endpoints (`dl.google.com/android/maven2/.../maven-metadata.xml`) and Maven Central's search API proved far more trustworthy than search-engine summaries or `mvnrepository.com` (which blocked fetches outright).
- **Bumped** (all confirmed-stable per Google Maven metadata): `androidx-activity` 1.10.1→1.13.0, `androidx-core` 1.16.0→1.19.0, `androidx-lifecycle` 2.9.1→2.11.0, `androidx-browser` 1.8.0→1.10.0, `chucker` 4.1.0→4.3.1 (the Kotlin-2.3.x-metadata incompatibility noted when Chucker was first pinned back in the original stack setup is confirmed resolved now that the project is on Kotlin 2.4.0).
- **`androidx-core:1.19.0` forced a `compileSdk` bump**: AAR metadata check failed, requiring `compileSdk 37`. Verified API 37 (Android 17) is a genuinely *stable* release (shipped June 2026, not a preview) before bumping — `android-compileSdk` 36→37. **`targetSdk` deliberately left at 36** — compileSdk and targetSdk are independent (compileSdk only exposes newer APIs at compile time; Android 17's actual behavior changes — mandatory adaptive-UI enforcement, `ACCESS_LOCAL_NETWORK` runtime permission, etc. — are targetSdk-gated, not compileSdk-gated), and Google isn't requiring targetSdk 37 until August 2027, so there's no reason to opt into that yet.
- **Explicitly left alone**, per the stable-only instruction:
  - `androidx-security-crypto` (1.1.0-alpha06) — not an alpha-vs-stable choice; the whole `EncryptedSharedPreferences` mechanism `SecureStorage.android.kt` is built on is now deprecated by Google in favor of DataStore + Tink. A real migration to plan separately, not a version bump — see the new [[Guide - App Store Launch Readiness]] P2 note.
  - `voyager` (1.1.0-alpha04) — genuinely unreliable/contradictory release-history data across sources; also the exact library just device-verified clean post-AGP-migration, so deliberately not touched right after that.
  - `haze` (1.7.2) — 2.0.0-alpha01 exists but is alpha with a breaking architectural change (blur split into a separate pluggable module); current 1.7.2 is itself the latest *stable* release of the 1.x line, so no change needed.
  - Koin, detekt, kotlinx-coroutines, kotlinx-serialization, SQLDelight — left pinned given the unreliable research above; recommend checking directly via Android Studio's version-catalog "Check for Updates" if precision is needed later.
- **Verified**: `shared:check composeApp:check androidApp:lintDebug shared:iosSimulatorArm64Test androidApp:assembleDebug` all clean (confirmed via test-result XML, zero failures). Device-verified on the Pixel 10 Pro XL: app launches fully populated (no regressions), Chucker's `chucker_transactions` notification channel confirmed live and intercepting traffic (proving 4.3.1 actually works, not just compiles), `SkillTrainingService` foreground service started cleanly at `targetSdkVersion:36`, zero fatal exceptions in logcat.
- Wiki: [[Guide - App Store Launch Readiness]] gains a P2 note about the `EncryptedSharedPreferences` deprecation.

## [2026-07-17] meta | Correction: security-crypto does have a stable 1.1.0 release
- The 2026-07-17 dependency-audit entry above claimed `androidx.security.crypto` was deprecated with "no longer receiving official support or stable releases." This was wrong — a web-search summary conflated the deprecation *announcement* (made at `1.1.0-alpha07`/`beta01`) with an actual end of releases. The real Google Maven metadata shows the line continued: `alpha07` → `beta01` → `rc01` → a genuine stable **`1.1.0`**, with the release notes confirming zero breaking changes from `alpha06`. Caught because the user asked to specifically double-check whether "only alpha available" cases might still have meaningful newer pre-releases worth taking — a fair challenge to the original research.
- Acted on it: bumped `androidx-security-crypto` to the real `1.1.0`, and `voyager` `1.1.0-alpha04` → `1.1.0-beta03` (same logic — real Maven Central timestamps show three later pre-releases exist in the same target-version line; deliberately did *not* jump to the chronologically-more-recent `1.0.1`, since that's an older major-version line that may lack APIs this project's code already depends on). Both verified: full suite green, then device-verified specifically — logout-capable SecureStorage path (existing encrypted refresh token decrypts fine under the new library and drives a real auto-login) and the full Voyager navigation surface (all 4 tabs + bottom sheet) again, given Voyager's documented AGP-compat history earlier the same day.
- [[Guide - App Store Launch Readiness]]'s P2 note corrected in place (not just appended) since it was actively misleading, per this vault's usual practice of fixing clearly-wrong content directly while logging the correction here.

## [2026-07-17] query | iOS Liquid Glass — still blocked, but architecturally not by library versions
- User asked to re-check whether iOS 26 Liquid Glass support is still blocked, recalling it used to be a library limitation. Read JetBrains' official doc (kotlinlang.org/docs/multiplatform/ios-liquid-glass.html, updated 2026-06-10) directly in the browser, not just a search summary.
- Finding: it was never really a "library version" problem and no dependency bump changes it — Liquid Glass is system-rendered through native `TabView`/`NavigationStack`/toolbar, which Compose Multiplatform's Skia-based rendering never touches. The only real path is a native SwiftUI navigation shell wrapping Compose-rendered screen content, a genuine architectural migration (not attempted — logged as backlog per user's explicit request).
- Pages read: [[Guide - App Store Launch Readiness]].
- New content: added as a P2 entry on [[Guide - App Store Launch Readiness]] (not a standalone page — user asked specifically for a P2 note).

## [2026-07-17] query | Monetization decision + donation compliance research
- Talked through monetization options with the user given CCP's licensing constraint (real-money IAP/subscriptions that gate access need separate written CCP authorization, already documented on [[Guide - App Store Launch Readiness]]). Ads considered and set aside (complexity + likely-poor revenue for a niche audience); "pay to remove ads" specifically identified as still being a real-money IAP that gates a feature (the ad-free experience), so it doesn't dodge the CCP-authorization requirement either. Landed on voluntary donations as the only pre-launch monetization worth pursuing.
- Researched US compliance for donations: 1099-K threshold is back to $20,000/200+ transactions for 2025/2026 (not the lower $600 figure from since-rolled-back legislation), though the underlying income is taxable regardless of whether a form gets filed. Separately, found a real store-policy wrinkle: Apple's external-payment-link allowance for donations (Guideline 3.2.1(vii)) explicitly excludes the "Games" category — unverified whether a game-companion utility app like this gets classified as Games or not, which matters for whether an external Ko-fi/GitHub-Sponsors link would even be permitted on iOS, vs. needing to fall back to Apple's own in-app tip via StoreKit.
- Pages read/updated: [[Guide - App Store Launch Readiness]] (Monetization P2 bullet extended, new Donations compliance bullet added).

## [2026-07-17] query | Confirmed: donation income isn't a tax-free gift
- Quick follow-up to the monetization/donation research above — user asked to confirm donations are just taxable income. Confirmed and sharpened the existing tax note on [[Guide - App Store Launch Readiness]]: not a gift under IRS rules specifically because it's tied to a product people use (same reasoning IRS applies to Patreon/Ko-fi/Twitch), not a separate new finding.

## [2026-07-19] dev | "Plasma conduit" neon-outline treatment — Artifact mockup → real Compose code
- **Trigger**: user shared a reference screenshot of a sci-fi neon-outline UI they liked (refined from "isometric 3D" to specifically "thin glowing wireframe lines"), asked for 3 Minmatar-colored Artifact mockup variants, picked variant 2 ("Plasma Conduit"), asked to see it run across all 5 faction themes, then asked to implement it for real.
- **Plan-mode scope decisions** (both confirmed directly with the user before writing code):
  - Icons: `EveIcons` are filled EVE-derived silhouettes, not stroke art. User chose to redraw as true open-stroke outlines over the lower-risk "keep fills, add glow halo" option.
  - Nav bar: `PodNavBar`'s existing Haze frosted-glass pill + sliding highlight stays exactly as built; only the icon glyphs inside it change.
- **Icon technique — reuse existing path data, don't redraw**: `ImageVector.Builder.addPath(...)` takes `stroke`/`strokeLineWidth`/`strokeLineCap`/`strokeLineJoin` independently of `fill`. All 5 `EveIcons` builders switched from `fill = Black` to `fill = null, stroke = Black` on their *original* path nodes — zero new geometry authored, full fidelity to the real EVE-derived art preserved. Confirmed on-device: all 5 render crisply at the 36dp nav bar size, including the most structurally complex one (`characterSheetIcon`'s hex/pilot mark).
- **`GlowCard`** (new, `ui/component/GlowCard.kt`) — drop-in `Card` replacement: `colorScheme.primary` border, 3 concentric outer alpha-fading rings drawn via `Modifier.drawBehind` *before* the shape clip (so they bleed past the card, same trick as the mockup's CSS `inset: -3px`), radial-gradient wash toward `colorScheme.surfaceContainer`. Replaced every real `Card` usage: Dashboard (`StatCard`, Training, Recent Activity), `PiScreen.PlanetCard`, `JobsScreen.JobCard`.
- **Glow technique — layered alpha rects, not `Modifier.blur`**: discovered `GradientProgressBar` is reused per-row inside `JobsScreen`'s scrolling list (not just a single Dashboard hero instance as first assumed), so real blur halos repeated per row were ruled out as an avoidable perf/cross-platform risk. Every glow (`GlowCard`, `GradientProgressBar`, the Dashboard avatar ring) is 2-3 extra `drawRoundRect`/`drawCircle` calls at falling alpha instead; text glow (ISK balance number) uses the real `TextStyle(shadow = Shadow(...))` API, not list-repeated.
- **No new `Theme.kt` tokens needed for glow**: every existing scheme's `onPrimaryContainer` is already a bright light-tint of that theme's `primary` — an exact match for the mockup's glow accent across all 5 themes. Only real addition: `AppTheme.gainColor` extension — universally green, amber on Gallente specifically (its own primary is emerald, so a fixed green collided with the theme's own accent). Consolidated two independently-hardcoded `0xFF3FB950` literals (Dashboard wallet rows, Jobs status chip/label) into this one source.
- **Deliberately out of scope**: `PiScreen`'s `PlanetTypeChip`/`StatusChip` colors (real EVE domain meaning — planet type, extractor status — independent of app theme) and `SecurityStatusBadge`'s high-security green (different semantic, same theoretical Gallente-collision risk, left as a known small follow-up).
- **Verified**: `ktlintCheck`/`detekt` clean (composeApp `detekt-baseline.xml` gained 3 new `@Preview`-function entries, matching this project's existing per-preview baseline convention). Device-verified on the connected Pixel: all 4 nav icons + settings gear crisp at real size, `GlowCard` + glow render correctly on both Ember (default) and Gallente, Gallente's `gainColor` amber swap confirmed in both Dashboard wallet rows and Jobs' "Complete" chip/"Ready to deliver" label, scrolled a 3-planet PI list (6 `GradientProgressBar` + 3 `GlowCard` instances on screen at once) with no visual issues. `selfcheck` pass: 0 findings.
- Wiki: [[ADR-017 - Neon Outline Card and Icon Treatment]] created; [[ADR-013 - Faction Color Themes]] gained a forward-pointing addendum; [[Screen - Dashboard]], [[Screen - PI]], [[Screen - Jobs]] `## Current UI` sections updated with fresh 2026-07-19 screenshots (also closed out `Screen - PI`'s stale "screenshot pending" placeholder left over from 2026-07-14, and fixed a stale claim there that the progress bar itself "turns green" on completion — it never did, only the status chip does); `index.md` updated.

## [2026-07-19] dev | GlowCard — two real glow-rendering bugs found via user visual review
- **Trigger**: after the dev entry above shipped, the user looked at the running app and said the card *edges* specifically weren't glowing, even though the ISK number and progress bar clearly were — asked where to fix it. First fix attempt (see below) wasn't right either; user said so directly ("не это имел ввиду", "look at the screenshots") rather than accept it, which is what surfaced the second, real bug.
- **Bug 1 — stroked rings are nearly invisible as "glow"**: `GlowCard`'s outer bleed rings used `style = Stroke(width = 1.dp.toPx())` — a thin 1dp outline at low alpha, imperceptible next to `GradientProgressBar`'s glow (already correct: **filled** rounded rects). Fixed by dropping `style` on every fake-glow draw call (defaults to `Fill`), including the Dashboard avatar ring which had the identical bug. First rebuild + device screenshot confirmed strong, visible glow bleeding outward from every card.
- **Bug 2 — found only because the user pushed back on the first fix**: that first rebuild also revealed a second, pre-existing defect the user's original report didn't quite name but was pointing at — a soft warm blob sitting in the dead center of the wide Training card and the tall Recent Activity card, unrelated to any content. Root cause: the card's interior background used `Brush.radialGradient(listOf(primary, cardColor))`, which glows brightest in the **center** — the opposite of the mockup's CSS `inset` box-shadow, which glows brightest **at the border**. A center-out radial gradient is a plausible-looking but backwards translation of an inset shadow; invisible on small/square elements (stat cards, where the center sits behind the ISK number anyway) but glaring on asymmetric ones. Fixed by replacing it with strokes centered on the border path itself, each half clipped away by the card's own shape clip so only the inward-facing half renders — the real Compose equivalent of an inset shadow.
- **Both fixes verified**: `ktlintCheck`/`detekt` clean, rebuilt + reinstalled + relaunched on-device after each of the two fixes (not just the final one), screenshotted both times. User confirmed the second fix directly: "вот сейчас оно выглядит как на макете" (now it looks like the mockup).
- **Environment note, not a bug**: mid-verification the connected device's theme was found switched to Amarr (not something this session's commands did) and then the adb connection dropped entirely — consistent with the user having picked up and used the physical phone independently while builds were running. Not investigated further since it's expected behavior for a real shared device, not a code defect.
- Wiki: [[ADR-017 - Neon Outline Card and Icon Treatment]] updated in place (not just appended) — the `GlowCard` decision paragraph was rewritten to describe the corrected technique instead of the original wrong one, and a new "Two real bugs found via visual review" section added with both root causes and the reusable rules of thumb (fake glow must be filled, not stroked; an edge glow needs a gradient anchored at the edge, not the shape's centroid). `Screen - Dashboard`'s `## Current UI` screenshot replaced with the corrected-glow version; `Screen - PI`/`Screen - Jobs` screenshots still show the pre-fix version since the device disconnected before they could be recaptured — flagged as a known small gap, not fixed in this entry.

## [2026-07-19] dev | Koin-in-@Preview crash — DashboardSuccess (pre-existing) + JobsSuccess (introduced today)
- **Trigger**: user pasted a real crash stack trace — `IllegalStateException: KoinApplication has not been started` at `DashboardScreen.kt:112` — and asked to check it.
- **Diagnosis**: the stack trace's lower frames (`android.view.View_Delegate`, layoutlib-style `RelativeLayout`/`LinearLayout` measurement, `com.android.internal.policy.DecorView`) are the signature of Android Studio's sandboxed `@Preview` renderer, not a real device or emulator — confirmed the real app was unaffected (it had just been screenshotted working fine moments earlier). Android Studio's Preview sandbox never runs the app's actual startup code (no `startKoin{}` from `PodForEveApplication.onCreate()`), so any composable reachable from a `@Preview` function that calls `getKoin().get()` directly throws this exact exception the moment the IDE tries to render it.
- **Scope, confirmed via `git blame`**: `DashboardSuccess`'s `val themeRepo: ThemeRepository = remember { getKoin().get() }` dates to commit `fe0de44a` (2026-07-10, the original [[ADR-013 - Faction Color Themes]] work) — pre-existing, not introduced this session. `JobsSuccess` gained the identical pattern *today* while wiring `gainColor` for [[ADR-017 - Neon Outline Card and Icon Treatment]], and `JobsPreviewSuccess` reaches it the same way `DashboardPreviewSuccess` reaches `DashboardSuccess` — a second, newly-introduced instance of the same bug class.
- **Fix**: `ThemeRepository.kt` gained `@Composable fun rememberThemeRepositoryOrNull(): ThemeRepository?` — `null` under `LocalInspectionMode.current` (the standard Compose signal for "currently being rendered by IDE tooling, not a real app"), the real Koin-resolved repo everywhere else. Both `DashboardSuccess` and `JobsSuccess` switched to it; read sites fall back to a local `MutableStateFlow(AppTheme.EMBER)` when the repo is `null`, and `DashboardSuccess`'s theme-switcher write becomes a harmless no-op (`themeRepo?.current = theme`) under Preview.
- **Verified**: `ktlintCheck`/`detekt` clean, compiles for both Android and iOS targets, rebuilt + reinstalled + relaunched on the reconnected device — real app renders identically to before (Koin still resolves normally outside the Preview sandbox), confirming the guard doesn't change real runtime behavior at all.
- Wiki: [[ADR-013 - Faction Color Themes]] gained a second addendum documenting the gap in its own established `getKoin().get()`-in-Composables pattern and pointing future consumers at the new helper.

## [2026-07-20] query | Is Compose Multiplatform production-ready?
- User asked for an honest opinion on CMP's current maturity, prompted directly by living through today's AGP-plugin friction and version churn. Answered inline first (version lockstep tax is real, core shared-logic layer is solid, Liquid Glass is a hard architectural ceiling — not a verdict of "not ready," but "ready with a real recurring tooling-tax, not 'boring' the way native-only development is").
- User then clarified the project's actual purpose: PodForEve exists specifically to evaluate CMP's current state firsthand — hitting and understanding friction largely *is* the point, not an incidental cost. This reframes prioritization project-wide; recorded as a `project`-type Claude memory (`project_goal_cmp_evaluation.md`) so future sessions carry this context automatically.
- Given that reframing, the user asked for a dedicated page consolidating these findings rather than leaving them scattered across ADR-012/ADR-016/ADR-017 and assorted log entries.
- New page: [[Guide - Compose Multiplatform Maturity Findings]] — synthesizes version-lockstep churn, the AGP 9 plugin transition's rough edges, the Liquid Glass platform-parity ceiling, third-party ecosystem lag (unreliable version-research, alpha-stuck libraries), and a fair "what's actually solid" section, closing with an overall assessment. Deliberately link-heavy per this vault's own "prefer linking over restating" rule — each finding points at the ADR/log entry with full detail rather than reproducing it.
- Pages updated: `index.md` (catalog entry), [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]] and [[ADR-016 - AGP KMP Library Plugin Migration]] (both gained a forward-pointing Reference to the new page, since they're its primary sources).
