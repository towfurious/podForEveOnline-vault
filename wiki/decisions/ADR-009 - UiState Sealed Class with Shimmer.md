---
title: ADR-009 — UiState Sealed Class with Shimmer
type: decision
tags: [adr, architecture, ui-state, layer-ui, compose-mp]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-009 — UiState Sealed Class + Modifier-Based Shimmer

## Status
Accepted 2026-04-24.

## Context
Every screen renders the same three conceptual states: waiting for first data, showing data, showing an error with retry. Without a shared shape, each screen invents its own flags (`isLoading: Boolean`, `error: Throwable?`, `data: Foo?`) and the rendering branches drift out of sync. We want exhaustiveness checking and a single shimmer implementation.

## Decision
Define a shared `UiState<T>` sealed class in `commonMain` with three branches — `Loading`, `Success(data)`, `Error(message, retry?)` — and route every screen through a single `UiStateHost` Composable. See [[UiState]] for the full contract.

Implement the **shimmer skeleton with `Modifier`** (a `drawWithContent` + animated-brush approach) — no third-party shimmer library. This keeps the dependency graph small and looks right on both platforms for free.

## Consequences
**Positive**
- One rendering contract across four screens.
- Compiler enforces exhaustive state handling via `when`.
- No shimmer library — one less dependency to track across KMP targets.
- `Success(staleData)` during background refresh composes cleanly with [[Stale-While-Revalidate Cache]]: we never regress to `Loading` once we have data.

**Negative**
- Doesn't natively encode "refreshing while showing stale" — we carry that as an auxiliary `isRefreshing: Boolean` on the per-screen state. Acceptable tradeoff vs. four UiState branches.
- If future screens need multi-resource states (e.g. primary + secondary data), we compose multiple `UiState<T>`s rather than introducing a product type. Keep it simple.

## Alternatives considered
- **`Flow<Result<T>>` + `isLoading` flag.** Rejected: less exhaustive, more ceremony per screen.
- **Accompanist / third-party shimmer.** Rejected: extra KMP concern; Modifier-based is ~30 lines.
- **UDF per-screen state classes with no shared base.** Rejected: loses the shared rendering host.

## References
- [[UiState]]
- [[Stale-While-Revalidate Cache]]
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
