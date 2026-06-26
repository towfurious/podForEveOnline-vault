---
title: Skill Queue
type: entity
tags: [entity, domain, skill, skill-queue, mvp]
aliases: [SkillQueue, Skill Training Queue]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Skill Queue

## Summary
The ordered list of skills the [[Character]] is training. Drives the Skills screen and the [[Screen - Dashboard|Dashboard]] "current training" widget. Live progress for the head of the queue is computed from timestamps on-device — see [[ADR-005 - Math-Based Skill Progress]] and [[Math-Based Progress Bar]]. No polling.

## ESI shape
**Endpoint:** `GET /characters/{character_id}/skillqueue/`
**Scope:** `esi-skills.read_skillqueue.v1` (plus `esi-skills.read_skills.v1` for base SP).

Each element:

```json
{
  "skill_id": 3301,
  "queue_position": 0,
  "finished_level": 4,
  "training_start_sp": 40000,
  "level_start_sp": 40000,
  "level_end_sp": 226000,
  "start_date": "2026-04-20T12:00:00Z",
  "finish_date": "2026-04-26T18:30:00Z"
}
```

`start_date` / `finish_date` may be `null` if the queue is paused or the skill won't start until a preceding one finishes.

## Business rules / invariants
- The **head** of the queue (`queue_position == 0`) is the currently training skill; only it has non-null dates in the normal case.
- **Paused queue** → all dates `null`. UI renders "Queue paused".
- Progress math: `now ∈ [start_date, finish_date]`, `progress = (now − start) / (finish − start)`, clamped to `[0, 1]`. SP gained = `level_start_sp + progress × (level_end_sp − level_start_sp)`.
- Refresh triggers: app foreground, pull-to-refresh, explicit queue change event (e.g. notification fired at completion).
- **Never poll** for progress updates — tick UI via coroutine, re-fetch only on known queue-shape changes.

## Lifecycle / state
- Fetched once on first display and cached in [[Stale-While-Revalidate Cache]].
- Stale cache serves instantly; background refresh updates it.
- On finish-date reached, notification fires (Android `ForegroundService` + `AlarmManager`; iOS scheduled `UNNotificationRequest`) — see [[ADR-006 - Android Foreground Service]] and [[ADR-007 - iOS Local Notifications]].
- Post-notification, re-fetch queue to learn new head.

## Related entities
- [[Character]] — owner.
- [[Math-Based Progress Bar]] — rendering pattern.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
