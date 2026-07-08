---
title: ADR-011 - Secrets via expect-actual and local.properties
type: decision
tags: [adr, auth, security, kmp, expect-actual, android, ios]
aliases: [ADR-011]
created: 2026-07-08
updated: 2026-07-08
sources: []
status: active
---

# ADR-011 — Secrets via expect/actual and local.properties

## Status
Accepted 2026-07-08

## Context
`EsiConfig.CLIENT_ID` was hardcoded as a string literal and committed to git. EVE SSO `client_id` is technically public (appears in every OAuth URL), but committing it is bad practice even for private repos — it prevents rotation, complicates open-sourcing, and violates hygiene rules. Needed a KMP-safe injection mechanism that works on both Android and iOS without a shared secret file.

History: the value `e82d24f308f2419e8840765769373c0f` was in git history; removed via orphan-branch force push before this ADR was written.

## Decision
Use `expect/actual` to inject the client ID platform-specifically:

```kotlin
// shared/commonMain
object EsiConfig {
    val CLIENT_ID: String get() = esiClientId()
    // ...
}
internal expect fun esiClientId(): String
```

**Android actual** (`shared/androidMain`) — reads from `BuildConfig`, which is populated from `local.properties` at Gradle evaluation time:

```kotlin
internal actual fun esiClientId(): String = BuildConfig.ESI_CLIENT_ID
```

```kotlin
// shared/build.gradle.kts
android {
    buildFeatures { buildConfig = true }
    defaultConfig {
        val localProps = Properties()
        rootProject.file("local.properties").takeIf { it.exists() }
            ?.let { localProps.load(it.inputStream()) }
        buildConfigField("String", "ESI_CLIENT_ID",
            "\"${localProps.getProperty("esi.client_id", "")}\"")
    }
}
```

`local.properties` (gitignored):
```properties
sdk.dir=/Users/vshavarin/Library/Android/sdk
esi.client_id=<your_client_id>
```

**iOS actual** (`shared/iosMain`) — reads from `Info.plist` via `NSBundle`, populated at build time by an xcconfig:

```kotlin
import platform.Foundation.NSBundle
internal actual fun esiClientId(): String =
    NSBundle.mainBundle.objectForInfoDictionaryKey("ESIClientID") as? String ?: ""
```

xcconfig file `iosApp/Configuration/Secrets.xcconfig` (gitignored):
```
ESI_CLIENT_ID = your_client_id
```
`Info.plist` contains `<key>ESIClientID</key><string>$(ESI_CLIENT_ID)</string>`.

## Consequences
- No secrets in source code or git history.
- Android CI needs `local.properties` injected as a secret env var or file.
- iOS CI needs `Secrets.xcconfig` injected.
- `BuildConfig` generation requires `buildFeatures { buildConfig = true }` in shared module's android block — not default for library modules.

## Alternatives considered
- **Hardcoded string** — rejected; see Context.
- **Gradle `buildConfigField` in `androidApp` only** — rejected; `esiClientId()` is called from `shared/commonMain` via `EsiConfig`, so the actual must live in `shared/androidMain`.
- **Environment variable at runtime** — not applicable to Android library modules without `BuildConfig`; iOS has no env vars at app runtime.

## References
- [[OAuth2 PKCE]] — context for why `client_id` matters.
- [[ADR-008 - OAuth2 PKCE via System Browser]] — the auth flow this feeds.
