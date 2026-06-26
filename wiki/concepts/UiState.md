---
title: UiState
type: concept
tags: [concept, architecture, layer-ui, ui-state, kmp, compose-mp]
aliases: [UI State, UiState sealed class]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# UiState

## Summary
A single sealed-class hierarchy representing the three states every screen goes through: **Loading**, **Success (with data)**, **Error (with message)**. Lives in shared KMP code; every ViewModel exposes a `StateFlow<UiState<T>>`; every screen has exactly one `when` over the three branches.

## Why here
Every [[Screen - Dashboard|screen]] in the app shows data fetched from ESI. The loading, error-retry, and content rendering logic is identical across all four screens. A shared `UiState<T>` gives us one shimmer implementation, one error-retry component, and one pattern a new contributor learns once. See [[ADR-009 - UiState Sealed Class with Shimmer]].

## Details

```kotlin
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String, val retry: (() -> Unit)? = null) : UiState<Nothing>()
}
```

### Rendering contract

```kotlin
@Composable
fun <T> UiStateHost(state: UiState<T>, onRetry: () -> Unit, content: @Composable (T) -> Unit) {
    when (state) {
        is UiState.Loading -> ShimmerSkeleton()
        is UiState.Success -> content(state.data)
        is UiState.Error   -> ErrorRetry(state.message, onRetry)
    }
}
```

- **Loading** → `Modifier`-based shimmer. No extra library — `Modifier.background(brush = …).drawWithContent { … }`.
- **Error** → inline message + retry button. No full-screen block unless there is no cached data at all.
- **Success** → full content.

### Transitions
`Loading → Success` on first successful fetch. `Loading → Error` on first failure. `Success → Success(updatedData)` on refresh. `Success → Success(staleData)` while background refresh is in flight (never regress to Loading once we have data — see [[Stale-While-Revalidate Cache]]).

## Tradeoffs
- **Pros:** exhaustiveness-checked by the compiler, one place to render every state, reusable across screens.
- **Cons:** doesn't model "refreshing while showing stale data" as its own variant — we thread that via a separate `isRefreshing: Boolean` flag on the containing ViewModel state instead of bloating `UiState`. Good enough for MVP.

## Related
- [[Stale-While-Revalidate Cache]] — why we don't regress Success → Loading.
- [[ADR-009 - UiState Sealed Class with Shimmer]].

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
