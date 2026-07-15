---
title: ADR-007 — iOS Local Notifications
type: decision
tags: [adr, platform, ios, unusernotificationcenter, bg-app-refresh-task]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-007 — iOS Local Notifications (no foreground-service equivalent)

## Status
Accepted 2026-04-24.

## Context
iOS has **no analog of Android's ForegroundService**. Third-party apps cannot run continuously to tick a live countdown in the lock screen or notification center. What iOS *does* offer: user-local notifications scheduled to fire at a specific date, and background-fetch windows the OS hands out at its discretion.

## Decision
- On queue fetch, schedule a `UNNotificationRequest` via `UNUserNotificationCenter.current().add(...)` for each upcoming skill completion at exact `finish_date`. Use `UNCalendarNotificationTrigger`.
- Cancel and reschedule on queue change (`removePendingNotificationRequests(withIdentifiers:)`).
- Register a `BGAppRefreshTask` (modern `BGTaskScheduler` API) to refresh the queue periodically. The OS decides when it runs — best-effort — so this is a belt, not a suspender.
- No live-countdown UI outside the app on iOS. Inside the app, the math-based progress bar (see [[ADR-005 - Math-Based Skill Progress]]) ticks every second as on Android.

## Consequences
**Positive**
- Accurate fire-at-completion notification without a long-running process.
- Uses documented, App-Store-safe APIs only.
- No daemon / no battery cost while the app is closed.

**Negative**
- **No live countdown** in the lock screen / Dynamic Island in MVP. Future work: a Live Activity with `ActivityKit` — out of MVP.
- `BGAppRefreshTask` is best-effort; the user may see noticeably stale data on re-entry if iOS has been stingy with background slots.
- All scheduled notifications must respect iOS's global 64-pending-notification limit — plenty for our 1–50 queued skills.

## Alternatives considered
- **VoIP push / Critical Alerts.** Rejected: requires permissions we're not entitled to and policy would reject the use case.
- **Third-party push (APNs from our server).** Rejected: we have no server — the app is client-only.
- **In-app only, no notifications.** Rejected: the whole point is "open the app only when there's something to do"; we need the pull.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[ADR-005 - Math-Based Skill Progress]]
- [[ADR-006 - Android Foreground Service]] — platform counterpart.
- [[Platform - iOS]]
- [[Skill Queue]]
- [[ADR-015 - Unified Completion Notifications]] — implements this design (2026-07-14) using `UNTimeIntervalNotificationTrigger`, and extends the same mechanism to industry jobs and PI extractors (all one-shot, same as skill queue here — iOS still has no live-countdown path).
