---
title: Platform - Android
type: platform
tags: [platform, android, foreground-service, alarm-manager, keystore, material3]
aliases: [Android]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Platform — Android

## Capabilities used
- **`ForegroundService`** for persistent training-countdown notification — see [[ADR-006 - Android Foreground Service]].
- **`AlarmManager`** exact alarms for skill-complete fires at `finish_date`.
- **Chrome Custom Tabs** (`CustomTabsIntent`) for OAuth login — see [[ADR-008 - OAuth2 PKCE via System Browser]]. Fallback: `Intent.ACTION_VIEW`.
- **`EncryptedSharedPreferences`** (Android Keystore) for refresh-token storage via [[SecureStorage]] `actual`.
- **Coil 3** for character portrait — see [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]].

## Permissions / entitlements
`AndroidManifest.xml` adds:

- `INTERNET` — Ktor calls ESI.
- `FOREGROUND_SERVICE` — required on API 28+.
- `POST_NOTIFICATIONS` — runtime-prompted on API 33+ (Android 13).
- `SCHEDULE_EXACT_ALARM` (or `USE_EXACT_ALARM`) — for the `AlarmManager` completion fire on API 31+ / 33+ respectively. Prefer `USE_EXACT_ALARM` (granted to calendar/alarm-like apps without a user grant).

Also:
- Intent filter for `eve-tracker://callback` on the OAuth entry activity.
- `<service android:foregroundServiceType="dataSync" ... />` for the training service.

## Lifecycle details
- App launch: init Koin, restore refresh token from `SecureStorage`, attempt silent token refresh. On success → foreground service starts if queue non-empty.
- App background → foreground: repositories refresh; `AlarmManager` already has the finish-date fire queued; nothing to re-wire.
- Process death: `AlarmManager` fire survives; service auto-starts via `BroadcastReceiver` → re-fetches queue → restarts tick loop.

## Gotchas
- OEM killers (Xiaomi, Huawei, some Samsung battery-saver profiles) can kill foreground services. `AlarmManager` is the safety net for the completion signal; the live-tick display is best-effort. Document this in the in-app help later.
- Android 14 tightened foreground-service types; `dataSync` still works for our "polls and ticks data" framing but watch for future target-SDK changes.
- `EncryptedSharedPreferences` has a known concurrency bug under heavy multi-thread writes on some OS versions — we only write the refresh token rarely so this is acceptable.
- `ASWebAuthenticationSession` equivalent on Android does not exist; treat Custom Tabs as close-enough but know cookies are shared with the user's default browser, which is usually what we want.

## Related
- [[ADR-006 - Android Foreground Service]]
- [[ADR-008 - OAuth2 PKCE via System Browser]]
- [[SecureStorage]]
- [[Platform - iOS]] — counterpart.
