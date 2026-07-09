---
title: Platform - iOS
type: platform
tags: [platform, ios, unusernotificationcenter, bg-app-refresh-task, keychain, xcode, compose-mp]
aliases: [iOS]
created: 2026-04-24
updated: 2026-07-08
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Platform — iOS

## Targets
- **Deployment target: iOS 16.0** — see [[ADR-010 - Platform Targets]].
- Compose Multiplatform 1.7+ stable iOS tier requires 16.0; we are aligned.

## Capabilities used
- **`UNUserNotificationCenter`** — schedule local notifications for skill completion at exact `finish_date`. See [[ADR-007 - iOS Local Notifications]].
- **`BGTaskScheduler` / `BGAppRefreshTask`** — best-effort periodic queue sync while backgrounded.
- **`ASWebAuthenticationSession`** — OAuth login, see [[ADR-008 - OAuth2 PKCE via System Browser]].
- **Keychain** (`Security.framework`) — refresh-token storage via [[SecureStorage]] `actual`.
- **Coil 3** — same KMP lib as Android for portrait rendering.

## Permissions / entitlements
`Info.plist`:
- `CFBundleURLTypes` with the `eve-tracker` scheme for OAuth callback.
- `BGTaskSchedulerPermittedIdentifiers` listing our background task identifier (e.g. `eve.tracker.queue.refresh`).
- `NSUserNotificationUsageDescription` — request runtime permission on first login.

Capabilities (Xcode):
- Background Modes → **Background fetch** (or **Background processing** if we ever move to `BGProcessingTask`).
- Push Notifications is **not** needed — we only send local notifications.

## Lifecycle details
- Launch: init Koin, restore refresh from Keychain, token-refresh. Register `BGTaskScheduler` handler and submit next refresh.
- Foreground: refresh repositories (queue, planets, jobs).
- Background: `BGAppRefreshTask` fires opportunistically; on success, re-schedule notifications for any newly changed `finish_date`s.
- App termination by user or OS: scheduled `UNNotificationRequest`s still fire; on re-launch we re-read everything.

## Kotlin/Native compilation gotchas (found 2026-07-08)
- **`GlobalContext` is JVM-only** — `org.koin.core.context.GlobalContext` does not exist in Kotlin/Native. All Composables that need Koin dependencies must use `org.koin.mp.KoinPlatform.getKoin().get()` instead. Affected: `App.kt`, `LoginScreen.kt`. `AuthCallbackHandler.kt` (shared/iosMain) already used the correct API.
- **`NSBundle` import required** — `platform.Foundation.NSBundle` must be explicitly imported; Kotlin/Native does not auto-import platform types. Found in `EsiClientId.ios.kt`.
- **`SecureStorage` keychain dictionary** — Security framework constants (`kSecClass`, etc.) are `CPointer<cnames.structs.__CFString>` in KN 2.0, not toll-free bridged to `NSString`. Must use `CFDictionaryCreateMutable` + `CFDictionarySetValue` (pure Core Foundation API) instead of `NSMutableDictionary`. See `SecureStorage.ios.kt`.
- **`xcode-select` must point to Xcode.app** — if `xcode-select -p` returns `/Library/Developer/CommandLineTools`, Kotlin/Native link step fails with `xcrun exit 72` ("unable to find utility xcodebuild"). Fix: `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`.

## Xcode project
- **Status: exists** — `iosApp/iosApp.xcodeproj` is set up and wired to KMP.
- Build phase: shell script runs `./gradlew :composeApp:embedAndSignAppleFrameworkForXcode`.
- Framework search paths: `composeApp/build/xcode-frameworks/$(CONFIGURATION)/$(SDK_NAME)`.
- Entry point: `iosAppApp.swift` calls `KoinInitKt.doInitKoin()` and routes `onOpenURL` to `AuthCallbackHandlerKt.handleAuthCallback(url:)`.
  - ⚠️ **`init`-prefix mangling** — Kotlin/Native ObjC codegen prepends `do` to any function whose name starts with `init`. `initKoin()` → `doInitKoin()` in generated header. Using the un-mangled name causes "method not found" crash at launch.

## Xcode build gotchas (resolved 2026-07-08)

All issues below were hit when first building the app in Xcode after the KMP scaffold was complete.

### 1. SQLite linker error
- **Symptom**: undefined symbols `_sqlite3_bind_blob`, `_sqlite3_open`, etc. at link time.
- **Cause**: `co.touchlab:sqliter-driver` bundles a SQLite C wrapper inside the static KMP framework. Static frameworks do not auto-propagate system library dependencies to the app target.
- **Fix**: add `OTHER_LDFLAGS = "-lsqlite3"` to both Debug and Release target build configurations in `project.pbxproj`.

### 2. `PBXFileSystemSynchronizedRootGroup` — "Multiple commands produce Info.plist"
- **Symptom**: Xcode 26 build error "Multiple commands produce `.../iosApp.app/Info.plist`".
- **Cause**: Xcode 26 uses `PBXFileSystemSynchronizedRootGroup` for new projects — it auto-syncs **all** files from a directory as both sources and resources. An `Info.plist` inside the synced folder is processed twice.
- **Fix**: place `Info.plist` in `iosApp/iosApp/Configuration/` (outside the auto-synced group). Update build settings: `GENERATE_INFOPLIST_FILE = NO`, `INFOPLIST_FILE = Configuration/Info.plist`.

### 3. Secrets injection — `ESI_CLIENT_ID` missing → "client_id parameter is required"
- **Symptom**: EVE SSO returns `{"error":"invalid_request","error_description":"The client_id parameter is required."}`.
- **Cause**: `EsiClientId.ios.kt` reads `NSBundle.mainBundle.objectForInfoDictionaryKey("ESIClientID")` but the key was absent from the auto-generated plist (which used `GENERATE_INFOPLIST_FILE = YES`).
- **Fix** (three steps):
  1. Create `iosApp/iosApp/Configuration/Secrets.xcconfig` (gitignored): `ESI_CLIENT_ID = <actual_id>`.
  2. Set `baseConfigurationReference` on both target configs in `project.pbxproj` to point at `Secrets.xcconfig`.
  3. In `Configuration/Info.plist` add `<key>ESIClientID</key><string>$(ESI_CLIENT_ID)</string>`. Xcode expands build-setting variables in Info.plist at build time (`INFOPLIST_EXPAND_BUILD_SETTINGS = YES` is the default).
- See also: [[ADR-011 - Secrets via expect-actual and local.properties]].

### 4. OAuth callback URL scheme — Safari "address is invalid"
- **Symptom**: after EVE SSO login, Safari shows "address is invalid" instead of returning to the app.
- **Cause**: custom URL scheme `eveauth-podforeve://` not registered; iOS cannot route it back to the app.
- **Fix**: add `CFBundleURLTypes` to `Configuration/Info.plist`:
  ```xml
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleURLName</key>
      <string>com.podforeve.tracker.iosApp</string>
      <key>CFBundleURLSchemes</key>
      <array><string>eveauth-podforeve</string></array>
    </dict>
  </array>
  ```

### 5. Compose Multiplatform `PlistSanityCheck` crashes
Compose 1.7.3 runs `PlistSanityCheck.performIfNeeded()` on a background GCD queue shortly after launch. It throws `kotlin.IllegalStateException` (uncaught → SIGABRT) on two distinct missing-plist-entry conditions.

**Check A — `UISceneConfigurations` empty dict**
- **Symptom**: SIGABRT on `com.apple.root.utility-qos` thread at launch.
- **Cause**: auto-generated plist (from `INFOPLIST_KEY_UIApplicationSceneManifest_Generation = YES`) produces `UIApplicationSceneManifest` with an empty `UISceneConfigurations: {}`. Compose rejects this.
- **Fix**: in the manual `Info.plist`, set `UIApplicationSceneManifest` to contain **only** `UIApplicationSupportsMultipleScenes = false`. Omit `UISceneConfigurations` entirely.

**Check B — `CADisableMinimumFrameDurationOnPhone` missing**
- **Symptom**: SIGABRT or exception shortly after launch (often triggered by first Compose render on high-refresh-rate simulator).
- **Cause**: Compose 1.7.3 requires this key to ensure ProMotion displays don't get capped at 60 fps.
- **Fix**: add to `Info.plist`:
  ```xml
  <key>CADisableMinimumFrameDurationOnPhone</key>
  <true/>
  ```
- **Verification**: binary search of linked dylib (UTF-16 strings) shows exactly **two** `PlistSanityCheck` error strings in CMP 1.7.3 — these two. No other required keys exist in this version.

## Gotchas
- **No live countdown** in lock screen / Dynamic Island. ActivityKit `Live Activities` would enable this — out of MVP scope (see [[ADR-007 - iOS Local Notifications]]).
- `BGTaskScheduler.submit` must be re-called at the end of each handler; forgetting it means the task never runs again.
- Global 64-pending-local-notifications cap — we're well under it, but don't schedule one per future skill; schedule only the current head (or head + next for robustness).
- `ASWebAuthenticationSession` cancellation must be caught — don't leave the UI in a limbo state.
- Compose Multiplatform on iOS: watch for recomposition performance on complex lists (PI, Jobs). If we hit issues, consider `LazyColumn` tuning before blaming Compose.

## Related
- [[ADR-007 - iOS Local Notifications]]
- [[ADR-008 - OAuth2 PKCE via System Browser]]
- [[ADR-011 - Secrets via expect-actual and local.properties]]
- [[SecureStorage]]
- [[Platform - Android]] — counterpart.
