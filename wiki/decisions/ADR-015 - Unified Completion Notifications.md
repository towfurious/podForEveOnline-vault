---
title: ADR-015 — Unified Completion Notifications
type: decision
tags: [adr, platform, android, ios, foreground-service, alarm-manager, unusernotificationcenter]
aliases: []
created: 2026-07-14
updated: 2026-07-15
sources: []
status: active
adr-status: Accepted
---

# ADR-015 — Unified Completion Notifications

## Status
Accepted 2026-07-14.

## Context
[[ADR-006 - Android Foreground Service]] and [[ADR-007 - iOS Local Notifications]] speced a skill-queue completion notification back in the original design spec — but neither was ever implemented (the manifest's `<service>` block sat commented out, zero notification code existed anywhere in the repo). [[Industry Job]] and [[Planet]] (PI extractors) had no notification design at all, despite carrying the same shape of data: a known future completion timestamp the user wants to be told about without having the app open.

## Decision
A single shared abstraction, `NotificationScheduler` (`expect`/`actual`, mirrors the [[SecureStorage]] pattern), with:
```kotlin
enum class NotificationSource(val idPrefix, val channelId, val channelLabel) { SKILL, INDUSTRY_JOB, PI_EXTRACTOR }
data class ScheduledCompletion(val id: String, val epochSeconds: Long, val title: String, val body: String)
expect class NotificationScheduler { fun reconcile(source: NotificationSource, items: List<ScheduledCompletion>) }
```
`reconcile()` cancels everything previously scheduled for that source and reschedules fresh from `items` — cancel-all-then-reschedule-all, no incremental diffing (dataset is tiny: one skill, a handful of jobs/planets). Called once from each of `SkillQueueRepository` / `IndustryJobRepository` / `PlanetRepository`, right after that repository's existing ESI fetch — using the freshly-fetched in-memory values directly, so **no new ESI calls and no DB migration were needed** (this includes `Planet`'s extractor expiry, which is never persisted to SQLDelight — scheduling reads it straight off the in-memory `ColonySummary` built during the same fetch).

**Android**: skill-queue training keeps the live-countdown ambition from ADR-006 — a `ForegroundService` (`SkillTrainingService`, lives in `shared/androidMain` since Gradle deps only flow `androidApp → composeApp → shared`) ticks the notification every second and self-detects completion, backed by an `AlarmManager` exact alarm as a kill-resilience fallback. Industry jobs and PI extractors get plain one-shot `AlarmManager` alarms (no foreground service, no live countdown — multiple simultaneous jobs/planets would make a persistent per-second ticker per instance pure clutter) via a shared `NotificationAlarmReceiver`. Two corrections vs. ADR-006's original wording:
- **`foregroundServiceType="specialUse"`, not `"dataSync"`.** targetSdk 35 (Android 15) caps `dataSync`-typed foreground services at 6 cumulative hours per rolling 24h window — fatal for skill training that can run for days. `specialUse` (API 34+, with a `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` manifest justification string) has no such cap.
- **`SCHEDULE_EXACT_ALARM`, not `USE_EXACT_ALARM`.** Google Play policy restricts `USE_EXACT_ALARM` to apps whose core function *is* alarms/timers/calendars. `SCHEDULE_EXACT_ALARM` is the general-purpose, revocable permission; `AlarmManager.canScheduleExactAlarms()` is checked at call time on API 31+, falling back to inexact delivery (`setAndAllowWhileIdle`) rather than crashing if not granted. Confirmed this isn't just theoretical: `adb install`-ing an APK update over an existing install logged `AlarmManager: Package com.podforeve.tracker, uid ... lost permission to set exact alarms!` — the grant gets revoked on update, not just on fresh install, so the inexact fallback is a real, regularly-hit path.
- Android has no API to enumerate pending `AlarmManager` alarms (unlike iOS), so `reconcile()` keeps its own bookkeeping of previously-scheduled ids per source in `SharedPreferences` to know what to cancel.
- All three of `SkillTrainingService`'s notification posts (the `startForeground()` anchor, the per-second tick update, and the backup alarm's completion post) share one fixed `Int` id (`SKILL_LIVE_NOTIFICATION_ID = 1001`, no string tag) so they land on the same slot — `ServiceCompat.startForeground()` only accepts a plain id, and a first pass that mixed a string tag for the ticker with the anchor's id/no-tag posted two permanent, non-dismissible notifications side by side instead of one updating notification. Caught visually on-device (two "Coherent Ore Processing" entries with different countdown values), not by code review.
- `ensureNotificationChannels()` is called from **both** `NotificationScheduler.init{}` and `SkillTrainingService.onCreate()` (idempotent, safe to call twice) — the Service doesn't only rely on the scheduler having constructed first. A first pass that only created channels in the scheduler's constructor crashed with `CannotPostForegroundServiceNotificationException` when the Service was reached through a path that didn't go through `NotificationScheduler` first (surfaced only via an abnormal invocation during verification, not normal app usage, but cheap enough to guard against unconditionally). `reconcile()` also wraps its whole body in a broad catch that logs and swallows rather than propagates — a scheduling failure must never take down the caller's data-fetch pipeline (repositories have their own unrelated try/catch around the ESI fetch; letting a `reconcile()` exception bubble into it silently skipped the "fresh data" emission too).

**iOS**: all three sources — including skill queue — are simple one-shot `UNNotificationRequest`s via `UNTimeIntervalNotificationTrigger` (not `UNCalendarNotificationTrigger`, to sidestep `NSDateComponents`/timezone/DST complexity for no benefit). No live countdown is possible here regardless of source, unchanged from ADR-007. `reconcile()` enumerates pending requests via `getPendingNotificationRequestsWithCompletionHandler`, filters by the source's id-prefix, and replaces them — no local bookkeeping needed (unlike Android), since the OS itself can enumerate what's pending.

**Permission request** lives outside `NotificationScheduler`, as a separate `@Composable expect fun RequestNotificationPermissionEffect()` (mirrors the existing `rememberHapticFeedback()`/`rememberUrlLauncher()` pattern), called once from `App.kt`'s `MainApp()`. Android's `POST_NOTIFICATIONS` runtime request needs an `Activity`-bound `ActivityResultLauncher`, which the shared-module actual (context-only, via `AppContext.instance`) has no way to obtain.

**Head-of-queue selection** (`SkillQueueRepository`) mirrors `DashboardViewModel`'s existing `activeSkill` selection — `freshEntries.firstOrNull { it.isTraining && !it.hasFinished(now) }` — **not** a bare `queuePosition == 0` check. Per [[Skill Queue]]'s documented ESI quirk, a just-finished skill can sit at `queuePosition == 0` for a while after its `finish_date` has passed (ESI doesn't immediately rotate it out); a naive position-0 check schedules a notification for an already-finished skill (silently dropped by the future-only filter) while missing the real, currently-training skill entirely. Caught during device verification (see log 2026-07-15), not by code review.

## Consequences
**Positive**
- All three completion types are covered with one shared contract; adding a fourth source later is a small, mechanical change.
- No new ESI calls, no DB migration — scheduling piggybacks on data the repositories already fetch.
- Android notification channels are per-source, so the user can mute one category via system Settings without silencing the others.

**Negative**
- Notifications only reflect whatever was true as of the last time the app was foregrounded — there is no background refetch (`BGAppRefreshTask` / `WorkManager`) in this pass, so a skill-queue reorder, a newly-queued job, or a newly-programmed extractor that happens while the app stays closed won't get scheduled until the next app open. Matches the literal ask ("notify me when it finishes") since starting any of these three things requires having the app open at some point after.
- Both the Android `ForegroundService` and its `AlarmManager` alarms are cleared on device reboot; there's no `BOOT_COMPLETED` receiver to re-arm them in this pass.
- Tapping a notification opens the app to wherever it was — no deep-link routing to the specific tab.
- Android's per-source `SharedPreferences` bookkeeping is new platform-side state to keep in sync; a bug here could leak a stale scheduled alarm (harmless — it would just fire a notification for an already-superseded item) or fail to cancel one (same failure mode).

## Alternatives considered
- **Foreground-service live countdown for all three sources.** Rejected — job/extractor can have multiple simultaneous instances (several jobs, several planets); a persistent per-second-ticking notification per instance is clutter, and multiplies Android 14+ foreground-service lifecycle risk for no benefit over a one-shot alert. Only the skill queue is a genuine "one singular thing happening right now."
- **`BGAppRefreshTask`/`WorkManager` periodic rescheduling in this pass.** Deferred — a real improvement, but a separate chunk of work (new Android dependency, non-trivial Swift/Kotlin bridging on iOS for the launch handler) solving a "belt not suspender" problem; the primary exact-time firing mechanism doesn't depend on it.
- **Persisting extractor expiry to SQLDelight to support scheduling from cold storage.** Rejected — scheduling only ever needs to happen synchronously right after a fetch, where the fresh in-memory value is already in hand; persisting it would be dead weight with no reader.

## References
- [[ADR-006 - Android Foreground Service]] — platform predecessor for skill queue; this ADR finishes implementing it and corrects the foreground-service-type/exact-alarm-permission wording.
- [[ADR-007 - iOS Local Notifications]] — unchanged in spirit; this ADR extends its scope to industry jobs and PI extractors.
- [[Skill Queue]], [[Industry Job]], [[Planet]] — the three data sources.
- [[SecureStorage]] — the `expect`/`actual` pattern this mirrors.
