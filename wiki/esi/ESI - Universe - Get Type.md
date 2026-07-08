---
title: ESI - Universe - Get Type
type: esi
tags: [esi, universe, skill, layer-data]
aliases: [GET /v3/universe/types/{type_id}/]
created: 2026-06-30
updated: 2026-06-30
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# ESI — Universe — Get Type

## Path + method
```
GET https://esi.evetech.net/v3/universe/types/{type_id}/
```

## Scopes required
None — public endpoint. Uses the `ssoClient` (unauthenticated) in `SkillQueueEsiApi`.

## Request shape
Path parameter: `type_id` (Int) — the EVE item type ID (same as `skill_id` in the skill queue response).

## Response shape (minimal — we only use `name`)
```json
{
  "type_id": 3300,
  "name": "Spaceship Command",
  "description": "...",
  "group_id": 257,
  "published": true
}
```

## Caching (ESI Cache-Control)
`Cache-Control: max-age=86400` (24 hours). We cache in the `skill_type` SQLDelight table indefinitely (skill names never change).

## Gotchas
- Skill names are stable — safe to cache permanently and never re-fetch unless the DB is migrated.
- Bulk resolution: for a queue of N skills, this is N sequential requests. Future optimisation: `POST /v2/universe/names/` accepts up to 1000 type IDs in one call.

## Maps to
[[Skill Queue]] (resolves `skill_id` → human-readable name), [[SQLDelight Migrations]] (stored in `skill_type` table).

## Related
- [[ESI - Skills - Get Skill Queue]]
- [[ESI Scopes MVP]]
