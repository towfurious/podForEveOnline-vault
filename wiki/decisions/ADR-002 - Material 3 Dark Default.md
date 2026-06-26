---
title: ADR-002 — Material 3 Dark Default
type: decision
tags: [adr, ui, material3, layer-ui]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-002 — Material 3 Dark Default

## Status
Accepted 2026-04-24.

## Context
The app needs a coherent look across Android and iOS. EVE Online's own aesthetic is dark, the audience plays at night, and the app is a glance-surface — high contrast in dim lighting matters. A full bespoke design system is overkill for 4 screens.

## Decision
Use **Material 3** as shipped by Compose Multiplatform, **dark color scheme by default**, with an `EveTheme` wrapper that injects EVE-palette accent colors on top of M3 tokens. A thin layer of primitives — `EveCard`, `ProgressSkillBar`, `StatusChip` — sits on M3 components and carries the `UiState` Loading/Success/Error rendering contract.

No custom design-system overhead: no new token taxonomy, no per-component theme spec. M3 is the design system.

## Consequences
**Positive**
- Zero design-system maintenance cost.
- Components like `Scaffold`, `NavigationBar`, `Card`, `CircularProgressIndicator` work out of the box.
- Dark-first means no low-contrast regressions in the primary use case.

**Negative**
- M3 aesthetic is recognizably Google on iOS — intentional tradeoff for shipping speed.
- Future theme toggling (light/system) is possible but not in MVP.

## Alternatives considered
- **Cupertino / adaptive theme per platform.** Rejected: doubles UI work, and the 5-second-glance goal is indifferent to platform-native feel.
- **Custom design system.** Rejected: not enough surface to amortize the work.
- **Light default.** Rejected: poor fit for EVE audience and use context.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[ADR-009 - UiState Sealed Class with Shimmer]] — shimmer rendering depends on M3 `Surface` tokens.
