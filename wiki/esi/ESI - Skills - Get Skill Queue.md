---
title: ESI - Skills - Get Skill Queue
type: esi
tags: [esi, skill, skill-queue, layer-data]
aliases: [GET /v2/characters/{character_id}/skillqueue/]
created: 2026-06-30
updated: 2026-07-08
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# ESI — Skills — Get Skill Queue

## Path + method
```
GET https://esi.evetech.net/v2/characters/{character_id}/skillqueue/
```

## Scopes required
`esi-skills.read_skillqueue.v1`

## Request shape
Path parameter: `character_id` (Long) — from JWT `sub` claim parsed by `EsiAuthService.extractCharacterId`.

No body. Access token attached automatically by Ktor auth interceptor.

## Response shape
Array of entries (empty array = queue is empty / paused):
```json
[
  {
    "queue_position": 0,
    "skill_id": 3300,
    "finished_level": 4,
    "start_sp": 45255,
    "finish_sp": 256000,
    "start_date": "2026-06-30T12:00:00Z",
    "finish_date": "2026-07-05T08:30:00Z"
  }
]
```
- `start_date` and `finish_date` are **absent** when the queue is paused (mapped to `null` in Kotlin).
- `start_sp` and `finish_sp` are skill points at the boundaries of the current level being trained.

## Caching (ESI Cache-Control)
ESI returns `Cache-Control: max-age=120` (2 minutes). Our SQLDelight cache stores `cached_at` but does not yet enforce TTL — we refresh on foreground + pull-to-refresh. Enforce TTL in a future iteration.

## Gotchas
- An empty array is valid and means the queue is empty (no skills training). Treat separately from a paused queue (entries present but `start_date`/`finish_date` null).
- **Completed entries are NOT immediately removed.** Entries with `finish_date` in the past remain in the response until ESI evicts them (timing unpredictable). Always filter client-side: `finishDate != null && finishDate <= Clock.System.now().epochSeconds`. Failing to filter causes the UI to show a completed skill with frozen 100% progress. See [[Skill Queue]] business rules.
- `finish_date` on the head entry is the source of truth for the Android `ForegroundService` countdown and iOS notification scheduling.
- `skill_id` is an EVE type ID — must be resolved to a name via `GET /v3/universe/types/{type_id}/`. See `SkillQueueRepository.resolveSkillName`.

## Maps to
[[Skill Queue]], [[SkillQueueRepository (code)]]

## Related
- [[ESI Scopes MVP]]
- [[ADR-005 - Math-Based Skill Progress]]
- [[Screen - Skills]]
- [[Platform - Android]] — AlarmManager fires at `finish_date`.
- [[Platform - iOS]] — UNNotificationRequest scheduled for `finish_date`.
