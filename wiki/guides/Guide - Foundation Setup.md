---
title: Guide - Foundation Setup
type: guide
tags: [guide, kmp, gradle, architecture, android, ios]
aliases: [Foundation Plan, Project Scaffold]
created: 2026-06-30
updated: 2026-06-30
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Guide — Foundation Setup

## What this covers
Scaffolding the Gradle KMP project in `podForEveOnline/` from a bare git repo to a compilable four-module skeleton.

## Module structure
```
podForEveOnline/
├── gradle/libs.versions.toml       — version catalog
├── settings.gradle.kts             — declares all modules
├── build.gradle.kts                — root plugin declarations
├── gradle.properties               — KMP / Compose / Android flags
│
├── shared/                         — domain + data (KMP library)
│   └── src/
│       ├── commonMain/             — Ktor, SQLDelight, Koin, domain models
│       ├── androidMain/            — OkHttp driver, EncryptedSharedPreferences
│       └── iosMain/                — Darwin driver, Keychain
│
├── composeApp/                     — shared UI + ViewModels (KMP library)
│   └── src/
│       ├── commonMain/             — Compose screens, Voyager, Coil3
│       ├── androidMain/            — tooling preview
│       └── iosMain/                — MainViewController (ComposeUIViewController)
│
├── androidApp/                     — Android entry point (application module)
│   └── src/main/
│       ├── AndroidManifest.xml     — permissions, ForegroundService stub, OAuth filter
│       └── kotlin/                 — MainActivity, Application, ForegroundService (future)
│
└── iosApp/                         — iOS entry point (Xcode project, not in Gradle)
    └── iosApp/
        ├── iOSApp.swift            — @main SwiftUI entry
        └── ContentView.swift       — UIViewControllerRepresentable → MainViewController
```

## Dependency graph
```
androidApp  →  composeApp  →  shared
iosApp (Xcode)  →  composeApp.framework  →  shared.framework
```

## Key decisions baked in
- [[ADR-001 - KMP Compose Multiplatform]] — stack choice.
- [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]] — library versions in `libs.versions.toml`.
- [[ADR-010 - Platform Targets]] — `minSdk = 28`, iOS deployment target = 16.0.
- [[SQLDelight Migrations]] — `AppDatabase.sq` is schema v1; `verifyMigrations = true`.
- [[SecureStorage]] — `expect class SecureStorage` stubs in `shared/platform/`.

## Files created
| File | Notes |
|---|---|
| `gradle/libs.versions.toml` | All versions — verify/bump before first build |
| `gradle/wrapper/gradle-wrapper.properties` | Gradle 8.9-bin |
| `settings.gradle.kts` | `:shared`, `:composeApp`, `:androidApp`; TYPESAFE_PROJECT_ACCESSORS |
| `build.gradle.kts` (root) | Plugin declarations, `apply false` |
| `gradle.properties` | JVM args, configuration cache, KMP flags |
| `shared/build.gradle.kts` | KMP library + SQLDelight plugin |
| `composeApp/build.gradle.kts` | Compose Multiplatform library |
| `androidApp/build.gradle.kts` | Android application module |
| `shared/.../UiState.kt` | Sealed class: Loading / Success / Error |
| `shared/.../SecureStorage.kt` | expect class + Android/iOS actuals (stubs) |
| `shared/.../AppDatabase.sq` | SQLDelight schema v1 (character, skill_queue_entry, planet, industry_job) |
| `composeApp/.../App.kt` | Root Composable, dark M3 theme |
| `composeApp/.../DashboardScreen.kt` | Voyager Screen stub |
| `composeApp/.../MainViewController.kt` | iOS ComposeUIViewController entry |
| `androidApp/.../MainActivity.kt` | ComponentActivity + setContent { App() } |
| `androidApp/.../AndroidManifest.xml` | Permissions, OAuth intent-filter, ForegroundService stub |
| `iosApp/iosApp/iOSApp.swift` | @main SwiftUI, embeds ContentView |
| `iosApp/iosApp/ContentView.swift` | UIViewControllerRepresentable wrapping MainViewController |

## Steps to first build

1. **Bootstrap Gradle wrapper** — the `gradle-wrapper.jar` binary is not in git. Run once from the project root:
   ```sh
   gradle wrapper --gradle-version 8.9
   ```
   Then use `./gradlew` for all subsequent commands.

2. **Open in Android Studio / IntelliJ IDEA** (2024.2+) — it will sync Gradle, download dependencies, index sources.

3. **iOS Xcode project** — the `iosApp/` directory has Swift sources but no `.xcodeproj` yet. Generate it:
   - Option A: In Android Studio, use "Open Xcode project" (KMP plugin integration).
   - Option B: Open Xcode → New Project → App → set "Deployment target: iOS 16.0" → save into `iosApp/` → add Compose framework embedding.
   - Option C: Run `./gradlew generateXcodeProject` if your KMP version supports it.

4. **Verify Android build**:
   ```sh
   ./gradlew :androidApp:assembleDebug
   ```

5. **Verify iOS framework build**:
   ```sh
   ./gradlew :composeApp:linkDebugFrameworkIosSimulatorArm64
   ```

## What is NOT done yet (next steps)
- Koin DI modules (shared data + platform modules wired up).
- ESI HTTP client setup (Ktor + auth interceptor).
- SQLDelight driver initialization for both platforms.
- Auth flow: OAuth2 PKCE — see [[OAuth2 PKCE]], [[ADR-008 - OAuth2 PKCE via System Browser]].
- All four screens beyond Dashboard stub.
- ForegroundService implementation — see [[ADR-006 - Android Foreground Service]].
- iOS notifications — see [[ADR-007 - iOS Local Notifications]].
