---
title: Screen - PI
type: screen
tags: [screen, pi, planet, pi-extractor, pi-factory, layer-ui, mvp]
aliases: [PI, Planetary Interaction, Planets]
created: 2026-04-24
updated: 2026-07-14
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

## Current UI

> _Screenshot pending — save the file to `attachments/` and replace this line with `![[screen-pi-2026-07-14.png]]`._

| Version | Date | Changes |
|---------|------|---------|
| v3 | 2026-07-14 | Legibility fix: secondary labels no longer stack alpha on top of `onSurfaceVariant`; smallest text raised to the app-wide 11sp floor |
| v2 | 2026-07-14 | Split SF/LP storage bars; pull-to-refresh; "data Xm ago" freshness label |
| v1 | 2026-07-09 | Initial implementation: colored PlanetTypeChip per EVE type (Barren=gold, Plasma=red, Storm=purple, Oceanic=blue), Attention/Idle status chips, cards with planet name + type + level + last-update time |

## Implementation notes
**File**: `composeApp/src/commonMain/.../ui/screen/PiScreen.kt`
- `PlanetCard` — card per planet; Column + StatusChip row layout.
- `StatusChip` — ACTIVE/NEEDS_ATTENTION/IDLE derived from `planet.status(now)`.
- `PlanetTypeChip` — colored pill for planet type; color pairs hardcoded per type (barren/plasma/storm/oceanic/temperate/lava/ice/gas). Falls back to `surfaceContainerHighest`/`onSurfaceVariant` for unknown types.
- `StorageRow(label, fillRatio, usedM3, capacityM3)` — called twice per colony card when capacity > 0: "Storage" row (SF, 12,000 m³ buffer for P0→P1) then "Launchpad" row (LP, 10,000 m³ finished-goods export).
- `PullToRefreshBox` wraps `PiSuccess`; spinner stays while ESI fetch is in-flight (SWR: stale cache emits first, fresh data arrives when fetch completes).
- "data Xm ago" label (11 sp, full `onSurfaceVariant`) shown per colony via `colony.dataAgeText(now)` — tells the user how stale the ESI data is (ESI colony endpoint has ~10-minute cache TTL).
- **Text floor convention**: no PI text goes below 11sp, and no text color drops below full-opacity `onSurfaceVariant` — matches the convention already followed on Dashboard/Jobs/Skills (`MaterialTheme.typography.labelSmall` and up, no `.copy(alpha=)` dimming). Icon tints are the one exception, deliberately kept at 0.6 alpha since they're decorative, not information.
- ViewModel: `PlanetViewModel` — exposes `isRefreshing: StateFlow<Boolean>`; `refresh()` sets it true, `loadPlanets()` clears it via `.onCompletion`.
- `ColonySummary.dataFetchedAtEpochSeconds` — epoch seconds captured immediately after `esiApi.fetchColony()` returns in `PlanetRepository`.

## Related
- ViewModel: `PiViewModel` (to create).
- Use case: `PiStatusUseCase`.
- Entities: [[Planet]], [[Character]].
- ADRs: [[ADR-009 - UiState Sealed Class with Shimmer]].
