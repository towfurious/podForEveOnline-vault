---
title: SQLDelight Migrations
type: concept
tags: [sqldelight, architecture, layer-data, kmp]
aliases: [Schema Versioning, DB Migrations]
created: 2026-06-30
updated: 2026-06-30
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# SQLDelight Migrations

## Summary
SQLDelight versions the database with an integer counter and numbered `.sqm` migration files. Version 1 = full MVP schema. Each breaking or additive schema change bumps the version and requires a corresponding migration file.

## Why here
[[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]] chose SQLDelight as the [[Stale-While-Revalidate Cache]] backing store. The spec deferred the exact versioning strategy; this page closes that thread.

## Details
- **Version 1** = the complete MVP schema: `character`, `skill_queue`, `planet`, `industry_job`, `wallet` tables.
- `.sq` files in `shared/src/commonMain/sqldelight/` define the schema and named queries.
- `migrations/` subdirectory holds `.sqm` files named by destination version (e.g. `2.sqm` = SQL to upgrade v1 → v2).
- Gradle config uses `verifyMigrations = true` so every migration path is compiled and validated at build time — migration errors surface before runtime.
- **Drop-and-repopulate policy**: because the store is a pure ESI cache (no user-authored data), a migration may `DROP TABLE` and `CREATE TABLE` if transforming old rows is complex. The app re-fetches on the next launch and treats it as a cold start.

## Tradeoffs
- Drop-and-repopulate is safe here precisely because no user data lives in SQLDelight — it is all re-derivable from ESI.
- `verifyMigrations = true` ensures no silent schema drift; prefer catching issues at build time.

## Related
- [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]]
- [[Stale-While-Revalidate Cache]]
- [[Character]], [[Skill Queue]], [[Planet]], [[Industry Job]]
