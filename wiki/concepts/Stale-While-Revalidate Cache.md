---
title: Stale-While-Revalidate Cache
type: concept
tags: [concept, architecture, layer-data, swr-cache, sqldelight]
aliases: [SWR, SWR Cache, Stale-While-Revalidate]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Stale-While-Revalidate Cache

## Summary
A cache discipline where reads **always serve whatever is cached immediately** — even if stale — and **trigger a background refresh** if the entry is older than its TTL. The UI never blocks on the network when there's any cached copy.

## Why here
EVE's ESI is cacheable (`Cache-Control: max-age=…` on most endpoints) and not particularly fast. Blocking a screen on a network round-trip is the enemy of a 5-second smoke-check app. Serving stale-and-refreshing means the user sees data instantly, the data gets fresh on screen shortly after, and we never punish them with a spinner when we could just show what we have. See also [[UiState]] for how screens render the mid-refresh state.

## Details

### The rule
- `Success(staleData)` is a valid render state — we keep showing it while refresh is in flight.
- `Loading` is **only** shown when the cache is *empty* (first-ever fetch).
- `Error` is only shown when a refresh fails **and** the cache is empty. Otherwise keep `Success(staleData)` and surface refresh failures less intrusively (snackbar, small badge).

### Where we implement it
- **Storage:** [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack|SQLDelight]] — typed queries over SQLite, KMP-native.
- **Repository pattern:** each repository exposes a `Flow<Data>` that emits from SQLDelight; repository also owns a `suspend fun refresh()` that hits Ktor and writes back. ViewModels subscribe to the flow and call `refresh()` on screen open and pull-to-refresh.
- **TTL:** per-entity, seeded from ESI `Cache-Control` / `Expires` headers. Override if the ESI TTL is uselessly short (some endpoints have sub-minute TTLs we don't need that aggressively).

### Refresh triggers
1. App foreground (from background → foreground transition).
2. Pull-to-refresh on any screen.
3. Known domain events — e.g. skill-complete notification fires → refresh [[Skill Queue]].

### What we **don't** cache
- EVE SSO tokens — see [[SecureStorage]].
- Short-lived domain-derived state (status chips, progress bars) — re-derived on render.

## Tradeoffs
- **Pros:** fast UX, fewer API calls, graceful under network loss.
- **Cons:** stale reads can mislead if we don't show freshness ("last updated X min ago" is nice for the user but optional in MVP); cache migrations need thought as the schema evolves (SQLDelight schema versioning).

## Related
- [[UiState]] — render contract for Loading / Success / Error including stale.
- [[Skill Queue]], [[Planet]], [[Industry Job]], [[Character]] — entities that flow through the cache.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
