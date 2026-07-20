---
title: Screen - Jobs
type: screen
tags: [screen, jobs, industry-job, layer-ui, mvp]
aliases: [Jobs, Industry]
created: 2026-04-24
updated: 2026-07-19
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Screen — Jobs

## Goal
See all active research and manufacturing jobs with live progress and time remaining. Answer: "which blueprints finish soonest?"

## Components
A scrollable list of rows, one per active industry job:

- Job type chip: Manufacturing / ME Research / TE Research / Copying (see [[Industry Job]] activity id → label mapping).
- Blueprint name.
- Station / structure name (resolved via `/universe/stations/{id}/` or `/universe/structures/{id}/`).
- Progress bar — uses [[Math-Based Progress Bar]].
- Time remaining.

Rows sort by `end_date` ascending (closest-to-completion first).

## Data it needs
- [[Industry Job]] — active jobs for the [[Character]].
- Blueprint type resolver (from `blueprint_type_id`) — cached heavily.
- Station / structure name resolver — cached heavily.

## States
- **Loading**: 3–5 placeholder rows shimmer.
- **Success**: rendered rows. `Success(staleData)` acceptable.
- **Error**: inline + retry.
- **Empty**: "No active jobs".

See [[UiState]].

## Interactions
- Pull-to-refresh → force refresh jobs.
- Tap row → job detail (post-MVP).

## Current UI

![[screen-jobs-2026-07-19.png]]

| Version | Date | Changes |
|---------|------|---------|
| v2 | 2026-07-19 | "Plasma conduit" neon-outline treatment ([[ADR-017 - Neon Outline Card and Icon Treatment]]): `JobCard` → `GlowCard`; `JobStatusChip`'s complete-state color and the "Ready to deliver" label now use `currentTheme.gainColor` instead of a hardcoded green literal, so Gallente shows amber instead of a green that would collide with its own emerald primary |
| v1 | 2026-07-09 | Initial implementation: Complete chip (green), 100% green progress bar, "Ready to deliver" label, job type (TE Research) + run count |

## Implementation notes
**File**: `composeApp/src/commonMain/.../ui/screen/JobsScreen.kt`
- `JobCard` — card per job; uses `SkillProgressCalculator.snapshot(start, end)` (ticks every 60 s via `produceState`). Uses `GlowCard` (`ui/component/GlowCard.kt`) as of 2026-07-19.
- `JobStatusChip` — derives state from `job.status + snapshot.progress`:
  - `status == "active" && progress < 1.0` → "Active" (gold chip)
  - `status == "active" && progress >= 1.0` → "Complete" (gain-colored chip — green on every theme except Gallente, which is amber); label shows "Ready to deliver". Note: the progress bar itself has never had a distinct "complete" color — it's always the same `GradientProgressBar` fill (theme primary → glow tint as of 2026-07-19; previously a fixed ember gradient), the v1 table row above describing it as "turns green" was inaccurate.
  - any other status (delivered/cancelled/…) → capitalized status string (gray chip); card opacity 0.75
- Cards with `!isActive` are rendered at `alpha(0.75f)` to visually de-emphasize finished work.
- ViewModel: `IndustryJobViewModel` (koin `koinScreenModel`).

## Related
- ViewModel: `JobsViewModel` (to create).
- Entities: [[Industry Job]], [[Character]].
- Pattern: [[Math-Based Progress Bar]].
- ADRs: [[ADR-009 - UiState Sealed Class with Shimmer]].
