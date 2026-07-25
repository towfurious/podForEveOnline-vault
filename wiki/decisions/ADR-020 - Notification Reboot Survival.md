---
title: ADR-020 - Notification Reboot Survival
type: decision
tags: [adr, android, notifications, sqldelight]
aliases: [BootCompletedReceiver]
created: 2026-07-24
updated: 2026-07-24
sources: []
status: active
adr-status: Accepted
---

# ADR-020 — Notification reboot survival: re-derive from SQLDelight cache, not a persisted schedule

## Status
Accepted 2026-07-24.

## Context
[[ADR-015 - Unified Completion Notifications]] named this as an explicitly deferred gap: Android clears both the `ForegroundService` (skill training's live countdown) and every `AlarmManager` alarm (job/extractor one-shots, plus SKILL's own backup alarm) on device reboot, and nothing re-arms them. [[Guide - App Store Launch Readiness]] P1 #8 left it as an open decision — "just a decision: P1 or P2?" — reasoning that opening the app at all already re-arms everything via each repository's normal `observe*()`/`reconcile()` path, and EVE players habitually check in often. The user chose to close it out for v1 rather than defer.

## Decision
A manifest-registered `BootCompletedReceiver` (`shared/androidMain/.../platform/BootCompletedReceiver.kt`), triggered on `android.intent.action.BOOT_COMPLETED`, re-derives notification state **from whatever's already cached in SQLDelight** — no network call, no login required:

- Queries `skill_queue_entry` (all rows — after [[OAuth2 PKCE|logout's cache wipe]], this table only ever holds the currently-logged-in character's rows, so no per-character grouping is needed), maps to real `SkillQueueEntry` domain objects, and reuses the existing `List<SkillQueueEntry>.activeSkill(now)` extension — the exact same head-of-queue selector [[Skill Queue]]'s own business rules already document (not `queuePosition == 0`), not a re-implementation of it.
- Queries `industry_job` (`status = 'active'`), maps to real `IndustryJob` domain objects, and reuses the `IndustryJob.activityName` computed property for the notification body — same reasoning, no duplicated activity-id-to-name mapping.
- Both paths call the existing `NotificationScheduler.reconcile(source, items)` — the identical entry point `SkillQueueRepository`/`IndustryJobRepository` already call after a live fetch, so there is exactly one code path that turns "a list of completions" into scheduled alarms/foreground-service state, whether it's reached from a fresh ESI fetch or a cold-boot cache read.

**PI extractor notifications are deliberately NOT re-armed by this receiver.** Confirmed directly from the [[Planet]] wiki page's own documented business rule: colony/extractor data ("scheduled directly from the in-memory colony snapshot at fetch time") is **never persisted to SQLDelight** — `PlanetRepository`'s own code comment says the same. There is nothing cached to reconstruct a PI extractor completion from after a reboot; that gap is accepted and matches the original P1/P2 framing's own reasoning (the next real app open re-arms it via a live fetch).

Whole `onReceive()` body wrapped in try/catch (matches `NotificationScheduler.reconcile()`'s own `@Suppress("TooGenericExceptionCaught")` convention) — a boot-time reconciliation failure must never crash the receiver or block anything else.

## Consequences

**Positive**
- Zero new state to persist or keep in sync — reuses the exact tables, domain models, and selectors the rest of the app already trusts, rather than introducing a second "what should be notified" representation that could drift from the live one.
- `NotificationScheduler.reconcile()` being the single entry point from both a live fetch and a cold-boot cache read means any future bug fix to reconcile-time logic (backup alarms, notification IDs, etc.) automatically covers both callers.

**Negative / watch items**
- PI extractor notifications don't survive reboot — a real, accepted, and now explicitly documented gap (see Decision above), not an oversight.
- Android only delivers `BOOT_COMPLETED` to apps the user has already launched at least once since install (apps in the "stopped" state are excluded from most implicit broadcasts) — standard OS behavior, not something to work around; the user will always have opened the app at least once to log in before this matters.
- If the user reboots the device without ever having opened the app again after a *previous* logout, the SQLDelight tables are empty (logout wipes them) — `reconcile()` is called with empty lists either way, which is a harmless no-op (stops a non-running service, cancels already-reboot-cleared alarms). No special-cased "am I logged in" check was needed.

## Alternatives considered
- **Persist a dedicated "next notification" table** (id/epoch/title/body per source) written every time `reconcile()` runs, read back verbatim on boot: rejected — duplicates the exact same data already sitting in `skill_queue_entry`/`industry_job`, and would need its own invalidation/cleanup story (e.g. on logout) kept in sync with the real cache by hand. Re-deriving from the tables that are already the source of truth avoids a second copy that can drift.
- **`WorkManager` periodic/boot-triggered work** instead of a `BroadcastReceiver`: rejected as unnecessary complexity for a one-shot, synchronous, sub-second SQLite read on a small (single-character) dataset — no background execution window or retry semantics are actually needed here.
- **Also persist PI colony/extractor data to survive reboot**: rejected — out of scope for this pass; would mean reversing [[Planet]]'s own documented "colony details are always fresh, never persisted" design for the sole benefit of a rare reboot-timing edge case, when the existing "next real fetch re-arms it" behavior was already judged acceptable by the very P1/P2 framing that motivated this ADR.

## References
- [[Guide - App Store Launch Readiness]] — P1 #8.
- [[ADR-015 - Unified Completion Notifications]] — where this was first named as a deferred gap.
- [[Skill Queue]] — the head-of-queue selector reused here.
- [[Planet]] — the documented reason PI extractor data can't be reconstructed.
