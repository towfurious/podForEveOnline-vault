---
title: Guide - App Store Launch Readiness
type: guide
tags: [guide, android, ios, mvp]
aliases: [Launch Readiness, Store Launch Checklist]
created: 2026-07-15
updated: 2026-07-16
sources: []
status: active
---

# Guide — App Store Launch Readiness

## Goal
Track everything PodForEve needs before a real Google Play / Apple App Store launch, beyond the app's core feature set (which is done and working — 4 screens, OAuth2 login, completion notifications, all committed and tested). Built 2026-07-15 from a full audit of the vault, the codebase, and current external store/CCP requirements. This is a **living tracker** — update the Status column as items get picked up, don't let it go stale.

## How to read this
Three tiers:
- **P0** — blocks store submission or is a hard compliance requirement. Nothing here should be skipped.
- **P1** — strongly recommended before a real public launch; not a hard blocker, but a real risk or quality gap.
- **P2** — legitimate backlog; fine to ship without.

Each row notes whether Claude Code can implement it directly, or whether it needs **your own action** first — an account, a credential, a hosting or wording choice only you can make.

## P0 — store submission blockers

| # | Item | Status | Why | Technical note | Requires your action? |
|---|---|---|---|---|---|
| 1 | Android release signing | Not started | Release builds are currently **unsigned** — no `signingConfigs` block anywhere in `androidApp/build.gradle.kts`; Play Console rejects unsigned bundles outright. | Add `signingConfigs.release` reading a gitignored `keystore.properties`, mirroring [[ADR-011 - Secrets via expect-actual and local.properties]]'s `local.properties` pattern exactly. Fail the Gradle build loudly if the file is missing, rather than silently falling back to unsigned/debug signing. | **Yes** — generate the actual `.jks` keystore file and decide where its passwords live. This is a durable secret: if it's ever lost, the app can never be updated again under the same listing. Not something to generate without you directly involved. |
| 2 | iOS app display name | Not started | Resolves to the literal Xcode target name **"iosApp"** — no `CFBundleDisplayName` / `INFOPLIST_KEY_CFBundleDisplayName` set anywhere. Android correctly shows "PodForEve". | Set `INFOPLIST_KEY_CFBundleDisplayName = PodForEve` in both Debug/Release configs of `iosApp/iosApp.xcodeproj/project.pbxproj` (or add `CFBundleDisplayName` directly to `Configuration/Info.plist`). | No — pure config fix. |
| 3 | CCP/EVE affiliation disclaimer | Not started | CCP's Developer License Agreement forbids implying official endorsement. No disclaimer exists anywhere in-app today. | Add a line near `LoginScreen.kt`'s existing "Your EVE Online companion" copy (`composeApp/.../ui/screen/LoginScreen.kt` lines ~57, 76) — standard fan-tool wording: *"EVE Online is a trademark of CCP hf. PodForEve is not affiliated with or endorsed by CCP hf."* Consider also surfacing it from a Settings/About location if one gets built (none exists yet — the theme-switcher bottom sheet from [[ADR-013 - Faction Color Themes]] is the natural place to add an About row). | Confirm the exact wording before shipping — a legal-risk-tolerance call, not a technical one. |
| 4 | Privacy policy | Not started | Both stores require a live privacy-policy URL at submission. None exists — no `LICENSE`/`PRIVACY`/ToS file anywhere in the repo. | Draftable directly: the app has no PodForEve-hosted backend — all character data comes live from CCP's ESI per-request and is cached only on-device (SQLDelight + EncryptedSharedPreferences/Keychain); nothing is sent to any server the developer controls; no analytics/tracking SDK exists today. That's a simple, honest policy to write. | **Yes** — choose where to host it (e.g. GitHub Pages off this repo is free and simple) and confirm publishing; publishing public content needs your go-ahead. |
| 5 | Play Data Safety form + Apple App Privacy nutrition label | Not started | Required questionnaires in each store's console before publishing — must accurately describe what data the app collects/transmits. | A reference doc (see item 4's data-flow summary) makes these fast to fill out accurately: "reads EVE character/skill/industry/PI data live from ESI per-request, caches it on-device only, transmits nothing to a PodForEve server, collects no analytics today." Revisit both forms the moment a crash reporter (P1 #4) is added — that changes the answer. | **Yes** — these are literal web forms inside Play Console / App Store Connect, only accessible to you. |
| 6 | Android targetSdk 36 | Not started | Google requires new submissions/updates to target API 36 (Android 16) by **August 31, 2026** — about 6 weeks from today (2026-07-15). App is on 35. [[ADR-010 - Platform Targets]] already has a "bump annually" policy but hasn't acted on this specific deadline yet. | Bump `android-targetSdk` (and `android-compileSdk`) in `gradle/libs.versions.toml`, then a full compile + device-verify pass — Android 16 behavior changes worth checking specifically: predictive back gesture, edge-to-edge enforcement. | No — pure code change, but should happen soon given the date. |
| 7 | Apple Privacy Manifest (`PrivacyInfo.xcprivacy`) | Not started | Likely required — SQLDelight's file access and/or the Kotlin/Native runtime itself plausibly trip the "required reason API" file-timestamp category, mandatory since May 2024. | JetBrains ships a purpose-built Gradle plugin for exactly this KMP scenario (`apple-privacy-manifests` — check kotlinlang.org's Compose Multiplatform privacy-manifest doc for current usage before wiring). Add it to `shared`'s iOS targets, verify the generated manifest bundles into the framework. | No — pure build-config change, but verify the plugin's current API since this wasn't independently confirmed beyond a search snippet. |

## P1 — strongly recommended before a real public launch

| # | Item | Status | Why | Technical note | Requires your action? |
|---|---|---|---|---|---|
| 1 | ESI `User-Agent` header | Not started | Confirmed absent by direct grep. CCP's ESI best-practices docs ask every client to identify itself (app name/version + contact) so CCP's ops team can reach the developer if something misbehaves. | `install(UserAgent) { agent = "PodForEve/<version> (<contact>)" }` on both Ktor clients in `PlatformModule` (`shared/src/androidMain/.../di/PlatformModule.android.kt` and the iOS counterpart). | **Yes** — what contact info (email or repo URL) you're comfortable exposing publicly. |
| 2 | ESI rate-limit / HTTP 420 handling | Not started | Confirmed absent. CCP tracks a per-client error budget via `X-Esi-Error-Limit-Remain`/`-Reset` response headers; exceeding it returns HTTP 420 on *all* endpoints for the rest of the window, and CCP's docs call repeated abuse bannable. | Add response-header-aware backoff to the Ktor client, and map HTTP 420 to a friendly "ESI is rate-limited, try again shortly" `UiState.Error` instead of a generic failure. | No — pure code change. |
| 3 | ESI cache-header compliance | Not started | Confirmed absent — no `Expires`/`ETag`/`Cache-Control` handling anywhere in the SWR cache layer. [[ADR-005 - Math-Based Skill Progress]] avoids *polling*, but doesn't gate individual trigger-based refetches (foreground/pull-to-refresh/notification) against ESI's own declared cache window — CCP's docs call refetching before `Expires` "cache circumvention," a bannable pattern. | Add conditional-request support (`If-None-Match`, honor 304, respect `Expires`) in the repository fetch logic. | No — pure code change. |
| 4 | Crash reporting | Not started | No crash-reporting or analytics SDK exists at all. Near-universal 2026 baseline expectation — currently zero visibility into production crashes. | Firebase Crashlytics (simpler if a Google account/project is fine) or Sentry (more portable, self-host option) — both have generous free tiers. Once wired, this SDK's data collection must be reflected in both the Play Data Safety form and Apple's nutrition label (P0 #5). | **Yes** — pick one and create the account/project; wiring the SDK happens after. |
| 5 | Account / data deletion | Not started | Play Store requires apps with "account creation" to offer in-app + web deletion; genuinely ambiguous whether EVE-SSO-only login counts, but the app does cache character-linked data locally, so the safer reading is to treat it as in-scope. | Likely simpler than it sounds — there's no PodForEve-hosted backend, so "delete my data" may just mean: (a) confirm the existing logout flow actually wipes the local SQLDelight cache and not just the auth tokens (needs checking), plus (b) a sentence in the privacy policy stating no server-side copy of user data exists anywhere, satisfying the "web-based deletion" half without building an actual web flow. | Possibly not, once (a) is confirmed — mostly a documentation + verification task. |
| 6 | Offline / connectivity detection | Not started | Confirmed absent anywhere in the app, despite being fully ESI-dependent. A dropped connection today likely surfaces as a generic `UiState.Error`. | Lightweight expect/actual connectivity check (`ConnectivityManager` on Android, `NWPathMonitor` on iOS) + a distinct "you're offline" state/banner. | No — pure code change. |
| 7 | CI/CD pipeline | **Done (2026-07-16)** — basic CI shipped | `.github/workflows/ci.yml` now runs on every push/PR to `master`: ktlint + detekt + Android Lint (all 3 modules) + `shared:testDebugUnitTest` + `androidApp:assembleDebug` (ubuntu-latest, debug APK uploaded as a build artifact), plus `shared:iosSimulatorArm64Test` (macos-latest, needed for the Kotlin/Native Xcode toolchain). Verified by running the exact command sequences locally first — both `BUILD SUCCESSFUL`. Needs **no secrets** — see [[ADR-011 - Secrets via expect-actual and local.properties]]'s 2026-07-16 addendum for why. | Still open: a **signed release-build job** is a separate follow-up, gated on P0 #1's keystore existing — that job would need the keystore (and possibly `local.properties`/`Secrets.xcconfig`) added as GitHub Actions secrets. If this repo is private, the macOS job costs ~10x the Actions minutes of the Ubuntu one; no action needed now, just worth knowing if minutes usage becomes a concern later. |
| 8 | Notification reboot survival | Not started | [[ADR-015 - Unified Completion Notifications]] already names this as deferred: Android's `ForegroundService` and `AlarmManager` alarms are both cleared on device reboot, nothing re-arms them. | A `BOOT_COMPLETED` receiver that re-triggers `NotificationScheduler.reconcile()` for whatever's currently cached — worth building only if this is judged launch-blocking rather than backlog (current read: probably fine to ship without, since opening the app at all re-arms everything, and EVE players habitually check in often). | Just a decision — P1 or P2? |

## P2 — backlog, fine to ship without

- **AGP 9 → 10 KMP-plugin migration** — [[ADR-012 - Stack Upgrade Kotlin 2.4 CMP 1.11 AGP 9]]'s bypass flags (`android.builtInKotlin=false`, `android.newDsl=false`) will be removed in AGP 10.0, but no firm deadline is known yet.
- **Localization / i18n** — the app is English-only today (zero string externalization, confirmed by grep); reasonable to stay that way for MVP given EVE's playerbase skews English-dominant. Revisit if there's real demand later.
- **Accessibility polish** — `Modifier.semantics` on custom composables (progress bars, status badges). The small existing icon/image surface area is already fully covered (`contentDescription` set correctly everywhere it appears today).
- **`CharacterOwnerHash` migration** — best practice for handling character-ownership transfers correctly (a character can be sold to a new owner; `characterId` alone would then silently point to different real-world data). Confirmed the app uses raw `characterId` as the identity key everywhere today (13 files). Real but low-frequency edge case; the fix is invasive (auth + all four repositories + DB schema) relative to how often it'd actually bite a given user.
- **Remaining [[ADR-015 - Unified Completion Notifications]] deferred items** — `BGAppRefreshTask`/`WorkManager` background rescheduling, tap-to-deep-link, in-app per-category notification settings toggle.
- **Monetization** — not currently planned. Worth documenting now regardless: CCP's Developer License Agreement permits ISK-based charges, voluntary (non-gating) donations, and ad-network revenue, but **real-money IAP or subscriptions that gate access require separate written CCP authorization**. If a paid tier is ever considered, that conversation with CCP needs to happen first.

## Vault hygiene fixed alongside this review
- [[ADR-013 - Faction Color Themes]] and [[ADR-014 - Ktlint Detekt Linter Setup]] recovered — `index.md` referenced both, neither file existed.
- Stale OAuth redirect-URI (`eve-tracker://callback` → actual `eveauth-podforeve://callback`) corrected across [[ADR-008 - OAuth2 PKCE via System Browser]] (addendum), [[Platform - Android]], [[Platform - iOS]], and [[OAuth2 PKCE]].

## Related
- [[ADR-015 - Unified Completion Notifications]]
- [[ADR-011 - Secrets via expect-actual and local.properties]]
- [[ADR-010 - Platform Targets]]
- [[ADR-008 - OAuth2 PKCE via System Browser]]
- [[ADR-013 - Faction Color Themes]]
- [[ADR-014 - Ktlint Detekt Linter Setup]]
