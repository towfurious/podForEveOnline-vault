---
title: Platform - iOS
type: platform
tags: [platform, ios, unusernotificationcenter, bg-app-refresh-task, keychain]
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
- Entry point: `iosAppApp.swift` calls `KoinInitKt.initKoin()` and routes `onOpenURL` to `AuthCallbackHandlerKt.handleAuthCallback(url:)`.

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
