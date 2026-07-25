---
title: ADR-021 - Firebase Crashlytics Integration
type: decision
tags: [adr, android, firebase, crashlytics, observability]
aliases: [Crashlytics]
created: 2026-07-24
updated: 2026-07-24
sources: []
status: active
adr-status: Accepted
---

# ADR-021 — Firebase Crashlytics integration (Android)

## Status
Accepted 2026-07-24.

## Context
[[Guide - App Store Launch Readiness]] P1 #4: no crash-reporting or analytics SDK existed at all — zero production crash visibility, a near-universal 2026 baseline expectation. User picked Firebase Crashlytics over Sentry on 2026-07-22 and, once Play Console identity verification cleared, asked Claude to create the Firebase project and wire it in directly.

A Firebase project ("Pod for Eve Online" / `pod-for-eve-online`) already existed under the user's account — created via the Firebase console (browser-driven, using the user's own logged-in session; Claude never entered any Google credentials) rather than a fresh one, avoiding a duplicate. It already had one Android app registered under a different, unrelated package name (`com.viktorshavarin.podforeveonline`) from some earlier, unconnected attempt — left alone; a second app entry was registered for this project's real `com.podforeve.tracker`.

## Decision
- **Android app registered** in the existing Firebase project as `com.podforeve.tracker`, nickname "PodForEve".
- **`androidApp/google-services.json`** — downloaded from the Firebase console and committed to the repo (not gitignored). Firebase's own documentation states this file is safe to commit (it's a public client identifier + a Firebase API key restricted by Firebase's own security rules and package/SHA checks, not a secret comparable to the release keystore) — and this repo is private regardless.
- **Gradle wiring** (`gradle/libs.versions.toml`, root `build.gradle.kts`, `androidApp/build.gradle.kts`): `com.google.gms.google-services` (4.5.0) and `com.google.firebase.crashlytics` (3.0.7) plugins, both applied only in `androidApp` (the module with the real `applicationId` and `google-services.json` — `shared`/`composeApp` don't need either). `firebase-bom` (34.16.0) platform dependency + `firebase-crashlytics` library, versions resolved via the exact latest stable releases confirmed directly against Google's Maven metadata (`dl.google.com/dl/android/maven2/.../maven-metadata.xml`), not guessed.
- **No manual initialization code** — the Crashlytics SDK auto-registers itself as the process's uncaught-exception handler via a self-installing `ContentProvider`, the standard Firebase pattern; nothing in `PodForEveApplication.onCreate()` needed changing.

## Consequences

**Positive**
- Closes a real, confirmed gap with the industry-standard tool for exactly this — zero code changes needed beyond dependency wiring.
- Crashlytics auto-instruments the whole process once initialized, so crashes originating in `shared`/`composeApp` code are caught too, not just `androidApp`.

**Negative / watch items**
- iOS Crashlytics not wired — deliberately deferred with the rest of iOS work per the 2026-07-22 Android-first sequencing decision.
- Adding this SDK changes what the Play Data Safety form (P0 #5) and Apple's nutrition label must disclose — [[Guide - App Store Launch Readiness]] already flagged this dependency when the tool was first chosen; the form itself is still the user's own action.
- No custom non-fatal logging / breadcrumbs wired yet (`FirebaseCrashlytics.log(...)`, `recordException(...)`) — this pass only gets automatic fatal-crash capture, the baseline ask. A reasonable future enhancement, not done now (no evidence yet that it's needed).

## Verification
Real, not assumed:
- `androidApp:assembleDebug` and `androidApp:assembleRelease` both succeeded — the release build's R8 pass required no new proguard rules (Crashlytics ships its own consumer rules), and `uploadCrashlyticsMappingFileRelease` ran and completed, confirming the Crashlytics Gradle plugin actually authenticated against the real Firebase project and uploaded the deobfuscation mapping — not just a local build-config check.
- **Device-installed and launched** (Pixel 10 Pro XL): logcat confirmed `FirebaseCrashlytics: Initializing Firebase Crashlytics 20.1.0 for com.podforeve.tracker` and a real fetched remote-config response from Firebase's backend (`FirebaseSessions: Fetched settings: {...}`) — genuine network round-trip to Firebase, not just SDK presence. No crash on launch; real character dashboard data rendered normally afterward.
- Not verified this pass: an actual forced test crash confirmed appearing in the Crashlytics dashboard (the standard Firebase-recommended validation step) — the evidence above (successful init, successful settings fetch, successful mapping upload) was judged sufficient without also waiting out a full crash-report-upload-and-dashboard-propagation cycle. Worth doing once before the actual Play Store submission if a stronger guarantee is wanted.

## Alternatives considered
- **Sentry**: rejected 2026-07-22, before this ADR — see `log.md` 2026-07-22 for the comparison (Firebase judged simpler given the user was already setting up a Google identity for Play Console).
- **Creating a brand-new Firebase project** instead of reusing the existing "Pod for Eve Online" one: rejected — a matching project already existed under the user's account; creating a second, differently-named one would just fragment things for no benefit.

## References
- [[Guide - App Store Launch Readiness]] — P1 #4.
- [[ADR-018 - ESI HTTP Essentials]] — this project's other recent Gradle-version-catalog-driven dependency addition, same "verify exact versions against the real registry" discipline.
