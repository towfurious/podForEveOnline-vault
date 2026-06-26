---
title: Platform - iOS
type: platform
tags: [platform, ios, unusernotificationcenter, bg-app-refresh-task, keychain]
aliases: [iOS]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Platform — iOS

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

## Gotchas
- **No live countdown** in lock screen / Dynamic Island. ActivityKit `Live Activities` would enable this — out of MVP scope (see [[ADR-007 - iOS Local Notifications]]).
- `BGTaskScheduler.submit` must be re-called at the end of each handler; forgetting it means the task never runs again.
- Global 64-pending-local-notifications cap — we're well under it, but don't schedule one per future skill; schedule only the current head (or head + next for robustness).
- `ASWebAuthenticationSession` cancellation must be caught — don't leave the UI in a limbo state.
- Compose Multiplatform on iOS: watch for recomposition performance on complex lists (PI, Jobs). If we hit issues, consider `LazyColumn` tuning before blaming Compose.

## Related
- [[ADR-007 - iOS Local Notifications]]
- [[ADR-008 - OAuth2 PKCE via System Browser]]
- [[SecureStorage]]
- [[Platform - Android]] — counterpart.
