---
title: ADR-024 - Automated Play Console Publishing
type: decision
tags: [adr, android, ci, gh-actions, play-console, versioning]
aliases: [ADR-024, Play Console Auto-Publish]
created: 2026-08-09
updated: 2026-08-09
sources: []
status: active
adr-status: Accepted
---

# ADR-024 — Automated Play Console Publishing

## Status
Accepted 2026-08-09.

## Context
[[ADR-023 - CI Release Keystore Debugging]] and the same-day [[ADR-011 - Secrets via expect-actual and local.properties]] addendum got `android-release` to a genuinely working state — a real signed AAB with a correctly-injected `client_id`. But getting that AAB from CI into testers' hands still required a manual download-and-upload through the Play Console web UI every time. Two more gaps stood in the way of automating that last step:

1. **`versionCode` was hardcoded to `1`** in `androidApp/build.gradle.kts`. Play Console rejects any second upload with a duplicate `versionCode` outright — the very first automated re-upload would have failed immediately.
2. **No CI credential existed for the Play Developer API at all.** Every previous secret ([[ADR-011 - Secrets via expect-actual and local.properties]], [[ADR-023 - CI Release Keystore Debugging]]) covers *building and signing* an artifact, not *publishing* one to Google.

## Decision
Two changes, both scoped to the existing manual-only (`workflow_dispatch`) `android-release` job in `.github/workflows/ci.yml` — this still never runs on push/PR, so an ordinary commit can't trigger a real publish.

**1. `versionCode` from the workflow's own run number.**
```kotlin
// androidApp/build.gradle.kts
versionCode = (project.findProperty("versionCode") as String?)?.toIntOrNull() ?: 1
```
CI passes `-PversionCode=${{ github.run_number }}` to `bundleRelease`. `run_number` increments on *every* run of the `CI` workflow (push, PR, or manual), not just release runs, so there are gaps in the sequence — but it's strictly increasing across the workflow's entire history, which is all Play Console requires. Local/debug builds fall back to `1` since Play Console never sees those.

**2. A new `Publish to Play Console` step**, using the `r0adkll/upload-google-play@v1` GitHub Action, appended after `bundleRelease`:
```yaml
- name: Publish to Play Console (closed testing track)
  uses: r0adkll/upload-google-play@v1
  with:
    serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
    packageName: com.podforeve.tracker
    releaseFiles: androidApp/build/outputs/bundle/release/*.aab
    track: alpha
    status: completed
```
`track: alpha` is the Play Developer API's legacy identifier for the default "Closed testing" track PodForEve already publishes to (see [[Guide - App Store Launch Readiness]] P0 #9) — Play Console's UI renamed it, the API name didn't change. `status: completed` rolls the release out to 100% of that track's testers immediately, no separate manual "review and roll out" click — matches what was actually wanted (CI push = build reaches testers), not a staged draft.

**Credential**: a new Google Cloud project (`podforeve`) with the Android Publisher API enabled, a dedicated service account (`github-actions-play-publisher@podforeve.iam.gserviceaccount.com`) created in it, and its JSON key stored as the `PLAY_SERVICE_ACCOUNT_JSON` repo secret. Granted access in Play Console → Users and permissions → Invite user (the current Play Console UI folds API-access service accounts into the same "invite a user" flow as human users — there's no longer a separate "API access" page unless a Cloud project was already linked from an older setup, which this account didn't have). Permission scope: **"Release apps to testing tracks" only** — deliberately not "Release to production, exclude devices, and use Play App Signing", so this credential cannot promote to production or touch the open-testing/production tracks even if leaked or misused. Promoting a closed-testing release further is left as a manual, deliberate action in the Play Console UI (Testing → track → Promote release) — see [[Guide - App Store Launch Readiness]].

**Explicitly decided against wider automation**: the job stays `workflow_dispatch`-only rather than auto-publishing on every push to `master`. A live-testers audience is real people opting in continuously for 14 days ([[Guide - App Store Launch Readiness]] P0 #9's tester-recruitment requirement) — an ordinary merged commit silently reaching them without a deliberate trigger was judged too easy to regret.

## Consequences

**Positive**
- One manual `workflow_dispatch` trigger now does build → sign → version → publish end-to-end; no more manual AAB download/upload through the Play Console web UI.
- `versionCode` collisions are structurally impossible going forward — no more remembering to bump a hardcoded number before a release.
- The publishing credential is scoped as narrowly as Play Console allows (testing tracks only), limiting blast radius if the secret were ever exposed.

**Negative / watch items**
- `run_number`-derived `versionCode` has gaps (it also increments on non-release CI runs) and isn't human-readable as a "build count" — acceptable since Play Console only requires monotonicity, not contiguity, but worth knowing if `versionCode` is ever eyeballed as a proxy for "how many releases have shipped."
- The `podforeve` GCP project is separate from the `pod-for-eve-online` project used for [[ADR-021 - Firebase Crashlytics Integration]] — deliberately kept apart per the user's own account-organization preference (different underlying Google account), not a technical requirement. Two GCP projects to keep track of instead of one.
- `status: completed` means a triggered run is *live to real testers immediately* — there is no dry-run or draft mode in this pipeline. Anyone running the job needs to treat the trigger itself as the point of no return, not the Play Console review step (there isn't one for closed testing).
- Not yet exercised end-to-end at the time this ADR was written — see `log.md` 2026-08-09 for the first real run's outcome.

## Alternatives considered
- **Fastlane `supply`** — the other common Play-publish tool; rejected in favor of `r0adkll/upload-google-play` for being a single well-maintained GitHub Action with no Ruby toolchain to add to a Kotlin/Gradle CI image, given this is the project's only publishing need.
- **Hardcoding the manual `versionCode` bump** — rejected; the whole point of this ADR is removing a manual step that's already caused one near-miss (the Play Console screenshot showing `1 (0.1.0)` that prompted this work).
- **Auto-publish on every push to `master`** — rejected; see Decision's closing paragraph.
- **Reusing the `pod-for-eve-online` Firebase/GCP project** for the service account — offered, but the user preferred a fresh project on a different Google account; no technical reason favored either.

## References
- [[Guide - App Store Launch Readiness]] — P0 #9 (Play Console track this publishes to).
- [[ADR-011 - Secrets via expect-actual and local.properties]] — the `local.properties`/secret-injection pattern this extends.
- [[ADR-023 - CI Release Keystore Debugging]] — the signing pipeline this builds on top of.
- `.github/workflows/ci.yml` — `android-release` job.
