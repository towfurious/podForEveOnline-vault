---
title: Screen - Jobs
type: screen
tags: [screen, jobs, industry-job, layer-ui, mvp]
aliases: [Jobs, Industry]
created: 2026-04-24
updated: 2026-07-09
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

![[screen-jobs-2026-07-09.png]]

| Версия | Дата | Что изменилось |
|--------|------|----------------|
| v1 | 2026-07-09 | Первая реализация: JobStatusChip Complete (зелёный), зелёный прогресс-бар 100%, "Ready to deliver" label, тип работы (TE Research) + количество ранов |

## Implementation notes
**File**: `composeApp/src/commonMain/.../ui/screen/JobsScreen.kt`
- `JobCard` — card per job; uses `SkillProgressCalculator.snapshot(start, end)` (ticks every 60 s via `produceState`).
- `JobStatusChip` — derives state from `job.status + snapshot.progress`:
  - `status == "active" && progress < 1.0` → "Active" (gold chip)
  - `status == "active" && progress >= 1.0` → "Complete" (green chip); progress bar turns green; label shows "Ready to deliver"
  - any other status (delivered/cancelled/…) → capitalized status string (gray chip); card opacity 0.75
- Cards with `!isActive` are rendered at `alpha(0.75f)` to visually de-emphasize finished work.
- ViewModel: `IndustryJobViewModel` (koin `koinScreenModel`).

## Related
- ViewModel: `JobsViewModel` (to create).
- Entities: [[Industry Job]], [[Character]].
- Pattern: [[Math-Based Progress Bar]].
- ADRs: [[ADR-009 - UiState Sealed Class with Shimmer]].
