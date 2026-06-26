---
title: Screen - Skills
type: screen
tags: [screen, skills, skill-queue, layer-ui, mvp]
aliases: [Skills]
created: 2026-04-24
updated: 2026-04-24
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

## Implementation notes
*(filled during dev — VM, use cases, `SkillProgressCalculatorTest`)*

## Related
- ViewModel: `SkillQueueViewModel` (to create; spec calls for `SkillQueueViewModelTest`).
- Use case: `SkillProgressCalculator` (spec calls for `SkillProgressCalculatorTest`).
- Entities: [[Skill Queue]], [[Character]].
- Patterns: [[Math-Based Progress Bar]].
- ADRs: [[ADR-005 - Math-Based Skill Progress]], [[ADR-009 - UiState Sealed Class with Shimmer]].
