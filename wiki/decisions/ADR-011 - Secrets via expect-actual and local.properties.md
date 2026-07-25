---
title: ADR-011 - Secrets via expect-actual and local.properties
type: decision
tags: [adr, auth, security, kmp, expect-actual, android, ios]
aliases: [ADR-011]
created: 2026-07-08
updated: 2026-07-23
sources: []
status: active
---

# ADR-011 — Secrets via expect/actual and local.properties

## Status
Accepted 2026-07-08

> **Addendum (2026-07-16)**: the Consequences bullets below ("Android CI needs `local.properties` injected...", "iOS CI needs `Secrets.xcconfig` injected...") turned out broader than necessary. Both `esiClientId()` actuals degrade gracefully to `""` when their source file is absent (`shared/build.gradle.kts:64`'s `.takeIf { it.exists() }`; `EsiClientId.ios.kt` reads from the app's own `Info.plist` at *app* runtime, which the `shared` module's Kotlin/Native compilation never touches) — this only breaks real SSO login, not compilation or unit tests. The CI workflow added in [[Guide - App Store Launch Readiness]] (lint/detekt/Android Lint/unit tests/`assembleDebug` + iOS K/N tests) needs **no secrets at all**. Secret injection is still real, just narrower in scope than originally stated: only a *signed release build* job (once the [[Guide - App Store Launch Readiness]] P0 keystore item exists) would need `local.properties`/`Secrets.xcconfig`-equivalent values in GitHub Actions secrets, and only if that job needs a working SSO login rather than just a signed artifact.

> **Addendum (2026-07-22)**: [[Guide - App Store Launch Readiness]] P0 #1 (release signing) extends this exact pattern one more time, for a Gradle-level secret rather than a KMP `expect/actual` one. `androidApp/build.gradle.kts` reads a gitignored root-level `keystore.properties` (same location and file format idea as `local.properties`) to populate a `signingConfigs.release`:
> ```properties
> storeFile=androidApp/keystore/podforeve-release.jks
> storePassword=<store password>
> keyAlias=podforeve
> keyPassword=<key password>
> ```
> `androidApp/keystore/` is gitignored alongside `keystore.properties` itself — the actual `.jks` (a durable, unrecoverable secret if lost) is generated and placed by the user directly, not by any automated tooling. Unlike `local.properties`/`ESI_CLIENT_ID`'s silent `""` fallback, this one **fails loudly**: `gradle.startParameter.taskNames.any { it.contains("Release") }` gates a `check(keystorePropertiesFile.exists())` that only fires when an actual release-producing task is requested (`assembleRelease`, `bundleRelease`, …) — debug builds and the existing CI job (which never builds release) are completely unaffected, verified by running `androidApp:assembleDebug androidApp:lintDebug` clean with the file absent, then `androidApp:assembleRelease` and confirming it fails with the intended message rather than silently producing an unsigned artifact. The `android-release` job in `.github/workflows/ci.yml` (added same day) base64-decodes an `ANDROID_KEYSTORE_BASE64` secret to that same relative path and reconstructs `keystore.properties` from sibling secrets (`ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`) before running `androidApp:bundleRelease` — manual-only (`if: github.event_name == 'workflow_dispatch'`, never on push/PR), YAML-validated but **not yet runnable**: none of those 4 secrets exist in the repo yet, and the keystore itself doesn't exist locally either. Scaffolded ahead of the actual secret so it's ready the moment both exist — the user explicitly asked for CI to be building with the key once it exists, not just today.
>
> **Closing note (2026-07-23)**: the keystore now exists — user asked Claude to generate it directly (`keytool`, judged fine since it's new local cryptographic material, not entering an existing credential anywhere; password disclosed in full to the user for their own backup). `androidApp:assembleRelease` and `androidApp:bundleRelease` both verified end-to-end (`apksigner verify` confirms the output APK's cert matches the keystore). Found and fixed a real bug along the way — see [[Guide - App Store Launch Readiness]] P0 #1's 2026-07-23 update for the R8/Tink `proguard-rules.pro` gap. Still open: pushing the 4 secrets to GitHub Actions — left to the user (`gh` isn't installed on this machine; GitHub's web UI works with no install).

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
