---
title: Planet
type: entity
tags: [entity, domain, planet, pi-extractor, pi-factory, mvp]
aliases: [Planetary Colony, PI Colony]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Planet

## Summary
A single planetary-interaction (PI) colony belonging to the [[Character]]. Drives [[Screen - PI]]. Each planet is reduced to a one-line status — **Active / Needs Attention / Idle** — computed from extractor and factory-chain state so the user can smoke-check all colonies at a glance.

## ESI shape
**Endpoints:**
- `GET /characters/{character_id}/planets/` — list of planets with `planet_id`, `planet_type`, `solar_system_id`, `last_update`, `num_pins`, `upgrade_level`.
- `GET /characters/{character_id}/planets/{planet_id}/` — detailed layout: `pins[]` (extractors, factories, storage, launchpads), `routes[]`, `links[]`.

**Scope:** `esi-planets.manage_planets.v1`.

Extractor pins carry `extractor_details` with `cycle_time`, `qty_per_cycle`, `head_radius`, `product_type_id`, and `expiry_time`. Factory pins carry `factory_details.schematic_id`.

## Business rules / invariants
Status derivation (see also `PiStatusUseCaseTest` in spec testing section):

- **Active** — all extractors running (`now < expiry_time`) AND factory chains OK (inputs flowing per `routes[]`).
- **Needs Attention** — any extractor depleted (`now ≥ expiry_time`) OR any factory chain starved/stopped.
- **Idle** — all extractors depleted AND no active factory production.

Extractor states: **Running** (time remaining shown), **Depleted** (expired).
Factory-chain states: **OK**, **Stopped** (no production), **Starved** (missing input).

Planet status is **derived, never stored**; recompute on every render.

## Lifecycle / state
- Fetched on foreground refresh and pull-to-refresh per [[Stale-While-Revalidate Cache]].
- No write operations in MVP — read-only observability.

## Related entities
- [[Character]] — owner.
- [[Screen - PI]] — consumer.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
