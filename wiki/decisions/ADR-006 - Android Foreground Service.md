---
title: ADR-006 — Android Foreground Service
type: decision
tags: [adr, platform, android, foreground-service, alarm-manager]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-006 — Android ForegroundService for Training Countdown

## Status
Accepted 2026-04-24.

## Context
On Android we want a **persistent notification in the shade** with the currently training skill name and a **live countdown** ticking every second. The notification must survive app-swipe and fire accurately at `finish_date`. Background-job APIs (`WorkManager`) don't run every second, and plain scheduled notifications can't show a live timer.

## Decision
Run a **`ForegroundService`** whose ongoing notification is the visible countdown. The service:

- Starts on login + when a non-empty [[Skill Queue]] is detected.
- Holds a coroutine that recomputes remaining time every second from `finish_date` (math-only — see [[ADR-005 - Math-Based Skill Progress]]) and calls `NotificationManager.notify()` with the updated text.
- Registers an **`AlarmManager`** exact alarm at `finish_date` → fires the "skill complete" notification and triggers a queue re-fetch.
- On queue shape change (fetch shows a different head) → cancel and reschedule the alarm.
- Stops when the queue empties.

Service type in manifest: `android:foregroundServiceType="dataSync"` (the most appropriate non-user-initiated type for this class of work on modern Android).

## Consequences
**Positive**
- Live countdown visible in the shade — matches spec's UX goal.
- Exact alarm for completion survives doze (via `setExactAndAllowWhileIdle` / `setAlarmClock` as appropriate).
- No polling — the ticking is math-only and CPU-trivial.

**Negative**
- Ongoing-notification UX can annoy users; we'll make it dismissible in a later iteration via a settings toggle that stops the service.
- Android 13+ requires `POST_NOTIFICATIONS` runtime permission.
- Android 14+ constrains foreground-service types — `dataSync` may require additional user-initiation justification in future versions; keep an eye on Android target-SDK changes.
- Battery-saver / vendor-killed-service behavior on some OEMs (Xiaomi, Huawei) remains the worst case; the `AlarmManager` backup still fires the completion notification.

## Alternatives considered
- **Plain `AlarmManager` with no live countdown.** Rejected: spec explicitly wants the live timer in the shade.
- **`WorkManager` periodic.** Rejected: minimum interval is 15 min, can't drive a per-second tick.
- **In-app ticker only.** Rejected: dies the moment the user leaves the app.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[ADR-005 - Math-Based Skill Progress]]
- [[Platform - Android]]
- [[Skill Queue]]
- [[ADR-015 - Unified Completion Notifications]] — implements this design (2026-07-14), with two corrections: `foregroundServiceType="specialUse"` (not `"dataSync"`, which caps at 6h/24h on Android 15) and `SCHEDULE_EXACT_ALARM` (not `USE_EXACT_ALARM`, a Play-policy risk for a non-alarm-app). Also extends the same mechanism to industry jobs and PI extractors.
