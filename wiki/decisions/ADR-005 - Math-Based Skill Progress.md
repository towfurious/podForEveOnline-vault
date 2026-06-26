---
title: ADR-005 — Math-Based Skill Progress
type: decision
tags: [adr, architecture, skill-queue, layer-domain]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-005 — Math-Based Skill Progress (no polling)

## Status
Accepted 2026-04-24.

## Context
The Skills screen and Dashboard need to show a live, second-ticking progress bar and a live countdown for the currently training skill. The naive approach — polling ESI every N seconds — is wasteful, hits ESI rate limits for no gain, and breaks in the background / with network loss.

ESI already returns everything we need: `start_date`, `finish_date`, `level_start_sp`, `level_end_sp`. The queue shape only changes on discrete events (skill finishes, user edits queue, pause/unpause).

## Decision
Fetch [[Skill Queue]] once, then compute progress **locally on every UI tick** using clock-based math:

```
progress = clamp((now - start_date) / (finish_date - start_date), 0.0, 1.0)
sp_now   = level_start_sp + progress × (level_end_sp - level_start_sp)
time_remaining = finish_date - now
```

A single coroutine on the Skills / Dashboard screen emits a tick every second; Composables recompose from the derived value. **No ESI call** per tick.

Re-fetch the queue only on:
- App foreground / pull-to-refresh (see [[Stale-While-Revalidate Cache]]).
- Notification fired (skill completed) — Android `ForegroundService` + `AlarmManager`, iOS scheduled `UNNotificationRequest`.

Same pattern applies to [[Industry Job]] progress bars — see [[Math-Based Progress Bar]].

## Consequences
**Positive**
- Zero per-tick network load. Works offline.
- Battery-efficient — no wake-lock for network, just CPU for a tiny math+recompose.
- Testable: `SkillProgressCalculatorTest` with injected `Clock` fully verifies the math.

**Negative**
- Client clock skew drifts the displayed progress. Acceptable: users tolerate a few seconds of inaccuracy on a multi-hour training; the notification at `finish_date` is the authoritative completion signal.
- If the queue actually changed server-side (e.g. user edited it on another device), the UI shows a stale picture until refresh. Mitigated by on-foreground refresh.

## Alternatives considered
- **Poll ESI every N seconds.** Rejected: wasteful and breaks without network.
- **Server-Sent Events / WebSocket from ESI.** Rejected: ESI doesn't offer this.
- **Poll with smart backoff.** Rejected: still pointlessly online when math + refresh-on-change is exact.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[Math-Based Progress Bar]]
- [[Skill Queue]], [[Industry Job]]
