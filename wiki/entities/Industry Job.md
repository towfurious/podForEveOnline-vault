---
title: Industry Job
type: entity
tags: [entity, domain, industry-job, blueprint, mvp]
aliases: [Industry, Manufacturing Job, Research Job]
created: 2026-04-24
updated: 2026-07-14
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Industry Job

## Summary
A single research or manufacturing job installed by the [[Character]]. Drives [[Screen - Jobs]]. Each job is displayed as a row with type, blueprint, station/structure, progress bar, and time remaining.

## ESI shape
**Endpoint:** `GET /characters/{character_id}/industry/jobs/?include_completed=false`
**Scope:** `esi-industry.read_character_jobs.v1`.

Each element (MVP-relevant fields):

```json
{
  "job_id": 123456,
  "activity_id": 1,
  "blueprint_id": 10000001,
  "blueprint_type_id": 999,
  "station_id": 60003760,
  "start_date": "2026-04-24T12:00:00Z",
  "end_date": "2026-04-25T18:00:00Z",
  "status": "active"
}
```

## Business rules / invariants
`activity_id` values we care about:

| id | type |
|---|---|
| 1 | Manufacturing |
| 3 | Time-Efficiency Research (TE) |
| 4 | Material-Efficiency Research (ME) |
| 5 | Copying |

Other activity ids exist (invention, reactions) but are out of MVP scope; render them with a generic label if they appear.

Progress math is identical to [[Skill Queue]]: `progress = (now − start_date) / (end_date − start_date)`, clamped `[0, 1]`. No polling — see [[Math-Based Progress Bar]] and [[ADR-005 - Math-Based Skill Progress]].

Blueprint names and station/structure names resolve through separate ESI calls (`/universe/types/{id}/`, `/universe/stations/{id}/` or `/universe/structures/{id}/`) — cached aggressively.

## Lifecycle / state
- Fetched on foreground refresh per [[Stale-While-Revalidate Cache]].
- `status` transitions from `active` → `delivered` once `end_date` is passed; we filter to active jobs only in MVP.
- On `end_date` reached, a one-shot completion notification fires for each active job (Android `AlarmManager`; iOS `UNNotificationRequest`) — see [[ADR-015 - Unified Completion Notifications]].

## Related entities
- [[Character]] — owner.
- [[Screen - Jobs]] — consumer.
- [[ADR-015 - Unified Completion Notifications]] — completion notification behavior.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
