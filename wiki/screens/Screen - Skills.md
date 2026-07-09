---
title: Screen - Skills
type: screen
tags: [screen, skills, skill-queue, layer-ui, mvp]
aliases: [Skills]
created: 2026-04-24
updated: 2026-07-08
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Screen — Skills

## Goal
Understand current training status and what's queued after it. Answer: "when does this finish, and what's next?"

## Components
- Large progress bar for currently-training skill (uses [[Math-Based Progress Bar]] — math only, no polling).
- Headline: skill name + target level.
- Two time labels: "Time to finish this skill" and "Time to finish full queue".
- Scrollable queue list — each row: skill name, level, duration.
- Paused-queue empty state: banner explaining the queue is paused + CTA to open EVE client.

## Data it needs
- [[Skill Queue]] (complete list, not just head).
- A domain `SkillProgressCalculator` (see [[Math-Based Progress Bar]] example).

## States
- **Loading**: shimmer list of 5–7 placeholder rows + big progress-bar skeleton.
- **Success**: full progress bar + list. `Success(staleData)` acceptable.
- **Error**: inline error + retry.
- **Paused queue**: a Success variant with a banner; the progress bar rendering is disabled / muted.

See [[UiState]].

## Interactions
- Pull-to-refresh → force refresh (invalidate cached queue, re-fetch).
- Tap a queue row → optional detail sheet (post-MVP).

## Current UI

![[screen-skills-2026-07-09.png]]

| Версия | Дата | Что изменилось |
|--------|------|----------------|
| v1 | 2026-07-09 | Первая реализация: hero bar "Coherent Ore Processing → Level 4  5d 10h 20m", очередь из 25+ скиллов с номерами и временем (формат Dd Hh Mm), Total в конце |

## Implementation notes

**ViewModel:** `SkillQueueViewModel` (Voyager `ScreenModel`) in `composeApp/commonMain/ui/viewmodel/`.
- Uses `refreshTrigger: MutableStateFlow<Int>` + `flatMapLatest` to re-subscribe to `SkillQueueRepository.observeSkillQueue` on pull-to-refresh.
- `uiState: StateFlow<UiState<List<SkillQueueEntry>>>`, `SharingStarted.WhileSubscribed(5_000)`.

**Use case:** `SkillProgressCalculator` in `shared/commonMain/domain/usecase/`.
- Injected `Clock` (defaults to `Clock.System`) for full testability.
- `snapshot(entry)` → `Snapshot(progress: Double, remaining: Duration)`.
- Returns `null` when entry is paused (no startDate/finishDate).
- Extension `Duration.formatHms()` → "Hh Mm Ss" string.

**Composables:**
- `SkillsScreen.kt` — top-level `Screen`; handles Loading/Success/Error/Empty/Paused states.
- `ActiveSkillProgressSection` — hero progress bar; `produceState` ticks every 1 s. No network per tick.
- `SkillQueueRow` — compact row; ticks every 60 s (minute-level accuracy sufficient for queue tail).
- `Shimmer.kt` — infinite alpha animation (`Modifier.shimmer()`), no extra library.
- `PullToRefreshBox` (M3 experimental) wraps the whole content area.

**Navigation:** `App.kt` — `TabNavigator` with M3 `NavigationBar`, 4 `Tab` objects. PI and Jobs are stub screens.

**Tests:**
- `SkillProgressCalculatorTest` (7 cases): zero at start, half at midpoint, one at end, clamping before/after, null when paused, `formatHms` coverage.

**Completed-entry filtering (fixed 2026-07-08):**
ESI returns all queue entries including ones with a `finish_date` in the past (already completed but not yet evicted from the response). Without filtering, `entries.first()` would show a finished skill with 100% progress frozen at 1.0.

Fix: `SkillQueueEntry.hasFinished(nowEpochSeconds: Long)` added to domain model:
```kotlin
fun hasFinished(now: Long): Boolean = finishDate != null && finishDate <= now
```
`SkillsScreen.kt` filters before selecting the head:
```kotlin
val now = Clock.System.now().epochSeconds
val pending = entries.filterNot { it.hasFinished(now) }
```
`DashboardViewModel.kt` applies the same filter when picking `activeSkill`.

**Queue row numbering (fixed 2026-07-08):**
After filtering completed entries, `queue_position` values are non-contiguous (e.g. 5, 6, 7 displayed as "5.", "6.", "7." when the real display positions should be "2.", "3.", "4."). Fixed by adding `displayPosition: Int` parameter to `SkillQueueRow` and passing `index + 2` from `itemsIndexed`.

**Key code paths:**
```
composeApp/commonMain/
  ui/screen/SkillsScreen.kt
  ui/viewmodel/SkillQueueViewModel.kt
  ui/component/SkillProgressBar.kt      ← displayPosition param added
  ui/component/Shimmer.kt
  di/AppModule.kt (uiModule)
shared/commonMain/
  domain/model/SkillQueueEntry.kt       ← hasFinished() added
  domain/usecase/SkillProgressCalculator.kt
  data/repository/SkillQueueRepository.kt
```

## Related
- ViewModel: `SkillQueueViewModel` (to create; spec calls for `SkillQueueViewModelTest`).
- Use case: `SkillProgressCalculator` (spec calls for `SkillProgressCalculatorTest`).
- Entities: [[Skill Queue]], [[Character]].
- Patterns: [[Math-Based Progress Bar]].
- ADRs: [[ADR-005 - Math-Based Skill Progress]], [[ADR-009 - UiState Sealed Class with Shimmer]].
