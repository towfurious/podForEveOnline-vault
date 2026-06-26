---
title: ADR-004 — Ktor SQLDelight Koin Coil3 Stack
type: decision
tags: [adr, architecture, ktor, sqldelight, koin, coil3, layer-data]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-004 — Ktor + SQLDelight + Koin + Coil 3 data/infra stack

## Status
Accepted 2026-04-24.

## Context
The shared layer needs: an HTTP client for ESI, a local store for the [[Stale-While-Revalidate Cache]], a DI container that works in Kotlin/Native and JVM, and an image loader for character portraits. Each choice must be KMP-native so the same code runs on Android and iOS without bridging.

## Decision
- **Ktor (client)** — ESI HTTP calls, content negotiation (`kotlinx.serialization`), auth interceptor for access-token attachment and 401 refresh.
- **SQLDelight** — typed SQL, generated KMP code, primary backing store for the SWR cache.
- **Koin** — dependency injection; declarative modules for `androidMain`, `iosMain`, `commonMain`.
- **Coil 3** — KMP-native image loading; character portraits URLs from EVE's images CDN.

## Consequences
**Positive**
- Each library is KMP-native and actively maintained.
- Ktor's plugin architecture cleanly handles auth token refresh and retry.
- SQLDelight's type-safe queries prevent runtime SQL-shape errors.
- Koin avoids annotation processors — fast incremental builds.

**Negative**
- Ktor error handling is lower-level than Retrofit — we write our own mapping into domain errors.
- SQLDelight schema migrations are manual (`NN.sqm` files).
- Koin discovery is runtime — no compile-time verification of the graph.

## Alternatives considered
- **Retrofit/OkHttp** — JVM-only; incompatible with iosMain.
- **Realm / Room KMP** — Room KMP was experimental at spec time; Realm adds weight and proprietary friction.
- **Kodein** — viable DI, but Koin's community and docs are larger.
- **Compose Kamel / Kamel** — image loaders that work, but Coil 3's KMP support and Compose integration are better.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[Stale-While-Revalidate Cache]]
- [[ADR-001 - KMP Compose Multiplatform]]
