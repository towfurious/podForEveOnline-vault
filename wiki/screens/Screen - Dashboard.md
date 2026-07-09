---
title: Screen - Dashboard
type: screen
tags: [screen, dashboard, layer-ui, mvp]
aliases: [Dashboard]
created: 2026-04-24
updated: 2026-07-09
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

## Current UI

![[screen-dashboard-2026-07-09.png]]

| Version | Date | Changes |
|---------|------|---------|
| v1 | 2026-07-09 | Initial implementation: 76dp portrait, corp name, security badge (gold +0.7), gear icon, ISK Balance + Skill Points stat cards, training card with progress bar, recent activity (Skill purchase / Market Escrow) |

## Implementation notes
**File**: `composeApp/src/commonMain/.../ui/screen/DashboardScreen.kt`

- `DashboardScreen` → `koinScreenModel<DashboardViewModel>()` → `DashboardContent(state, onRetry, onLogout)`.
- **Header (76 dp portrait)**: `AsyncImage` (Coil 3) + corp name + `SecurityStatusBadge` (colored pill: green ≥5.0, gold 0–5, red <0) + `IconButton(EveIcons.Settings)`.
- **Settings**: `ModalBottomSheet` (M3 experimental) toggled by `showSettings: MutableState<Bool>` via `remember { mutableStateOf(false) }`. Contains a single `TextButton("Log out", color=error)` that calls `onLogout()`.
- **Stats row**: 2-column `Row` with `StatCard` (ISK gold, SP neutral). SP shows "—" while `totalSp == 0L` (stale cache).
- **Training card**: reuses `ActiveSkillProgressSection` component.
- **Recent activity card**: shown only when `walletJournal.isNotEmpty()`. Up to 3 `WalletJournalRow`s with income/expense coloring (+green / neutral) and `relativeTime()` helper.
- **ViewModel** (`DashboardViewModel`): `combine(observeCharacter, observeSkillQueue, observeWalletJournal)` — 3-flow combine. `observeWalletJournal` emits `emptyList()` immediately so combine fires without blocking. `fun logout()` calls `screenModelScope.launch { authRepository.logout() }`.
- **No DB migration**: `CharacterInfo` gained `securityStatus/corporationName/totalSp` with defaults (0.0/""/0L). Stale DB rows serve defaults; fresh ESI refresh populates all.
- **ESI calls added** (no new scopes): `GET /v4/corporations/{id}/` (public), `GET /v4/characters/{id}/skills/`, `GET /v6/characters/{id}/wallet/journal/`.
- `EveIcons.Settings` added — Material Design gear path (24×24 viewport, EvenOdd fill for centre hole).
- BUILD SUCCESSFUL (compileDebugKotlinAndroid, 0 errors, only pre-existing @Preview warnings).

## Related
- ViewModel: `DashboardViewModel`.
- Entities: [[Character]], [[Skill Queue]], [[Planet]], [[Industry Job]].
- Pattern: [[Math-Based Progress Bar]].
- Domain model: `WalletJournalEntry` (new — `displayName` maps `ref_type` → readable string).
