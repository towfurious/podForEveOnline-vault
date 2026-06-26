---
title: Screen - PI
type: screen
tags: [screen, pi, planet, pi-extractor, pi-factory, layer-ui, mvp]
aliases: [PI, Planetary Interaction, Planets]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Screen — PI

## Goal
Scan all planetary colonies and know at a glance whether any need attention. Don't force the user to read numbers — let the **status chip** answer.

## Components
A scrollable list of one card per planet. Each card:

- Planet name + type chip (e.g. "Temperate IV") and solar system.
- Big status chip: **Active** (green) / **Needs Attention** (amber) / **Idle** (gray).
- Extractor row: Running (shows time remaining) / Depleted.
- Factory chains row: OK / Stopped / Starved.

Cards reorder by priority: Needs Attention first, then Active, then Idle — so the user's eye lands on problems.

## Data it needs
- [[Planet]] list — full detail per planet.
- Status logic is a pure use case (`PiStatusUseCase` — spec calls for `PiStatusUseCaseTest`).

## States
- **Loading**: shimmer skeleton of 3 placeholder cards.
- **Success**: cards rendered. `Success(staleData)` acceptable.
- **Error**: inline error + retry.
- **Empty**: "No planetary colonies configured" — only when user legitimately has no PI; link to EVE client.

See [[UiState]].

## Interactions
- Pull-to-refresh → force refresh planets.
- Tap card → planet detail sheet with full pin layout (post-MVP; MVP shows summary only).

## Implementation notes
*(filled during dev — VM, use cases, status logic, tests)*

## Related
- ViewModel: `PiViewModel` (to create).
- Use case: `PiStatusUseCase`.
- Entities: [[Planet]], [[Character]].
- ADRs: [[ADR-009 - UiState Sealed Class with Shimmer]].
