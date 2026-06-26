---
title: SecureStorage
type: concept
tags: [concept, auth, secure-storage, keystore, keychain, expect-actual, layer-platform]
aliases: [Secure Storage, expect class SecureStorage]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# SecureStorage

## Summary
A KMP `expect class SecureStorage` with platform `actual` implementations: **Android Keystore** (via `EncryptedSharedPreferences`) and **iOS Keychain** (via the Security framework). The only piece of data that flows through it in MVP is the EVE SSO **refresh token**.

## Why here
[[OAuth2 PKCE]] gives us a short-lived access token (≈20 min, kept in memory only) and a long-lived refresh token (must survive app restarts). The refresh token is the crown jewel — if leaked, an attacker can impersonate the character indefinitely until the user revokes access in the EVE developer portal. Storing it in plain `SharedPreferences` / `UserDefaults` is not acceptable; hardware-backed keystore is the right home. See [[ADR-008 - OAuth2 PKCE via System Browser]].

## Details

### Interface (commonMain)

```kotlin
expect class SecureStorage {
    suspend fun write(key: String, value: String)
    suspend fun read(key: String): String?
    suspend fun delete(key: String)
}
```

### Android (`androidMain`)
- `EncryptedSharedPreferences` with `MasterKey.Builder(...).setKeyScheme(AES256_GCM).build()`.
- Keys live in Android Keystore (hardware-backed where available; falls back to software on old devices).
- File-backed — survives reinstall only if user has cloud backup that includes app data; for our purposes, treat it as per-install.

### iOS (`iosMain`)
- Keychain items via `SecItemAdd` / `SecItemCopyMatching` / `SecItemDelete` from `Security.framework`.
- Access-control class: `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` — available after first unlock post-boot, not synced to iCloud, not migrated in device backup.

### Keys stored
- `eve.refresh_token` — the only key in MVP.

### What never goes here
- **Access tokens** — in-memory only, 20-min TTL, let them die with the process.
- **PKCE verifiers** — ephemeral per login, held in memory only.
- **Any ESI response data** — that belongs in [[Stale-While-Revalidate Cache]] (SQLDelight), not secure storage.

## Tradeoffs
- **Pros:** hardware-backed on both platforms, standard APIs, minimal surface.
- **Cons:** per-device only (no cloud sync) — user re-authenticates on a new device, which is actually the correct behavior. Migrations (e.g. key rotation) are custom code we'll write if ever needed.

## Related
- [[OAuth2 PKCE]] — token origin.
- [[ADR-008 - OAuth2 PKCE via System Browser]].

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
