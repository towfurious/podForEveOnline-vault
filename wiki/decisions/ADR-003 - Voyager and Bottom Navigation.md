---
title: ADR-003 — Voyager and Bottom Navigation
type: decision
tags: [adr, architecture, voyager, layer-ui, compose-mp]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-003 — Voyager + NavigationBar

## Status
Accepted 2026-04-24.

## Context
We need multi-screen navigation that works on both Android and iOS in Compose MP, with a bottom tab bar pattern and a graceful adaptation for tablets. Compose Multiplatform's own `androidx.navigation` was not yet fully multiplatform at the time of spec-writing, and we want a small, well-suited library — not a homegrown stack.

## Decision
Use **Voyager** for in-app navigation. Use Material 3 **`NavigationBar`** at the bottom on phones with **4 destinations**: Dashboard · Skills · PI · Jobs. Use **`NavigationRail`** on tablets / landscape for the same 4 destinations. Reserve a **5th slot** (unused in MVP) so future expansions — Assets, Mail, Market — don't require a navigation refactor.

## Consequences
**Positive**
- Voyager is KMP-native, small, ergonomic, supports nested navigators — enough for 4 tabs + per-tab stacks.
- `NavigationBar` is first-party M3, works on both platforms through Compose MP.
- The 5th-slot convention enforces forward-compat discipline without shipping dead UI.

**Negative**
- Voyager is a third-party dependency — we accept its maintenance pace.
- We can't use Android's `androidx.navigation-compose` type-safe routes; Voyager uses its own `Screen` class abstraction.

## Alternatives considered
- **`androidx.navigation-compose`.** Rejected: Android-only at spec time; Compose MP story incomplete.
- **Decompose.** Rejected: powerful but heavier than we need; its component tree model is a lot of ceremony for 4 tabs.
- **Hand-rolled state-hoisted navigator.** Rejected: low ceiling, reinvents back-stack handling badly.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[ADR-001 - KMP Compose Multiplatform]]
- [[ADR-002 - Material 3 Dark Default]]
