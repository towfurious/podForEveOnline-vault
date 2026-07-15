---
title: Planet
type: entity
tags: [entity, domain, planet, pi-extractor, pi-factory, mvp]
aliases: [Planetary Colony, PI Colony]
created: 2026-04-24
updated: 2026-07-14
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
- On extractor depletion (`extractorExpiryEpochSeconds` reached — the earliest expiry across a planet's extractor pins), a one-shot notification fires per planet (Android `AlarmManager`; iOS `UNNotificationRequest`). Scheduled directly from the in-memory colony snapshot at fetch time — this value is never persisted to SQLDelight. See [[ADR-015 - Unified Completion Notifications]].

## PI Structure Type IDs

Verified from live ESI colony data (2026-07-13). Each planet type has distinct type IDs for every structure.

### Command Centers (empty; no role in storage logic)
| Planet type | type_id |
|-------------|---------|
| Barren      | 2524    |
| Lava        | 2549    |
| Storm       | 2550    |
| Gas         | 2534    |
| Temperate   | 2254    |

### Extractors
| Planet type | type_id |
|-------------|---------|
| Barren      | 2848    |
| Lava        | 3062    |
| Storm       | 3067    |
| Gas         | 3060    |
| Temperate   | 3068    |

### Basic Industry Facilities (BIF)
| Planet type | type_id |
|-------------|---------|
| Barren      | 2473    |
| Lava        | 2469    |
| Storm       | 2483    |
| Gas         | 2492    |
| Temperate   | 2481    |

### Advanced Industry Facilities (AIF)
| Planet type | type_id |
|-------------|---------|
| Barren      | 2474    |
| Lava        | 2470    |
| Storm       | 2484    |

### Storage Facilities (12,000 m³)
| Planet type | type_id |
|-------------|---------|
| Barren      | 2541    |
| Lava        | 2558    |
| Storm       | 2561    |
| Gas         | 2536    |
| Temperate   | 2562    |

### Launchpads (10,000 m³)
| Planet type | type_id |
|-------------|---------|
| Barren      | 2544    |
| Lava        | 2555    |
| Storm       | 2557    |
| Gas         | 2543    |
| Temperate   | 2256    |

> Oceanic, Plasma, and Ice planet type IDs are TBD — no colonies on those planet types yet.

Detected via `schematic_id` (present on all factory pins regardless of planet type). Type IDs for factories vary per planet type and are **not** used for factory detection in code.

## Related entities
- [[Character]] — owner.
- [[Screen - PI]] — consumer.
- [[ADR-015 - Unified Completion Notifications]] — extractor-depletion notification behavior.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
