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
- Next: SkillQueueViewModelTest or Dashboard screen (ISK + training widget)..

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
