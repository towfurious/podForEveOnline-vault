---
title: Screen - Dashboard
type: screen
tags: [screen, dashboard, layer-ui, mvp]
aliases: [Dashboard]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Screen — Dashboard

## Goal
In 5 seconds the user knows: (1) am I logged in as the right character, (2) how much ISK do I have, (3) is something training, (4) does any PI colony or industry job want my attention. Open the app → answer → close.

## Components
- Top row: character portrait (Coil 3) + name + corporation name.
- ISK balance (right-aligned, monospaced, grouping separator).
- Current training card: skill name, level, mini progress bar (uses [[Math-Based Progress Bar]]), time remaining.
- Summary rows:
  - "N planets need attention" (tap → [[Screen - PI]]).
  - "N jobs completed / finishing soon" (tap → [[Screen - Jobs]]).
- Loading → shimmer skeleton per section.
- Error inline per section with retry.

## Data it needs
- [[Character]] — portrait, name, corporation, ISK.
- [[Skill Queue]] — head only.
- [[Planet]] — derived count with `status == NeedsAttention`.
- [[Industry Job]] — derived count of jobs nearing completion and completed.

Data flows via repositories over [[Stale-While-Revalidate Cache]]; ViewModel combines the four sources into a single dashboard state.

## States
- **Loading** (only when all caches empty): full-screen shimmer skeleton mirroring the layout.
- **Success**: rendered content; `Success(staleData)` acceptable while background refresh is in flight.
- **Error**: per-section inline error with retry. Never a full-screen error if *some* section has data.

See [[UiState]] and [[ADR-009 - UiState Sealed Class with Shimmer]].

## Interactions
- Pull-to-refresh → force refresh all four data sources.
- Tap current-training card → [[Screen - Skills]].
- Tap summary row → respective screen.
- Long-press portrait → account menu (logout) — post-MVP unless trivial.

## Implementation notes
*(filled during dev — code paths, files, ViewModel, tests)*

## Related
- ViewModel: `DashboardViewModel` (to create).
- Entities: [[Character]], [[Skill Queue]], [[Planet]], [[Industry Job]].
- Pattern: [[Math-Based Progress Bar]].
