---
title: ADR-013 — Faction Color Themes
type: decision
tags: [adr, architecture, material3, layer-ui]
aliases: []
created: 2026-07-10
updated: 2026-07-10
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-013 — Faction Color Themes

## Status
Accepted 2026-07-10.

> Reconstructed 2026-07-15 from the `log.md` entry recorded at the time — this page was never written up when the feature shipped, only referenced (broken) from `index.md`. Found during the [[Guide - App Store Launch Readiness]] audit.

## Context
[[ADR-002 - Material 3 Dark Default]] fixed a single dark M3 theme (`EmberColorScheme`, Minmatar-red) as the MVP default and explicitly deferred "future theme toggling" to later work. EVE Online's four playable factions (Minmatar, Amarr, Caldari, Gallente) each have a strongly recognizable color identity that players already associate with their characters/corporations — a natural, low-effort way to let users personalize the app beyond the single hardcoded scheme.

## Decision
- **`AppTheme` enum** (5 entries): `EMBER` ("Minmatar", the existing default), `AMARR`, `CALDARI`, `GALLENTE`, `AMOLED` (a pure-black bonus option, not faction-themed). Each exposes `toColorScheme()` → M3 `darkColorScheme` and a `previewColor` swatch for the picker UI.
- **Palettes**: Minmatar = existing ember red; Amarr = gold `#D4A820` on purple-black `#0A0608`; Caldari = ice blue `#4888E0` on dark navy `#06080E`; Gallente = emerald `#30B858` on near-black green `#060A07`; AMOLED = electric blue `#4090FF` on pure black `#000000`.
- **`ThemeRepository`** (new, `composeApp/commonMain`) wraps a `MutableStateFlow<AppTheme>` backed by a `SecureStorage` key (`app.theme`), so the selection survives restarts. Read via `KoinPlatform.getKoin().get()` directly in Composables — no `koin-compose` dependency was added just for this.
- **Switcher UI**: the Dashboard gear button opens one `ModalBottomSheet` with two pages — a menu page (Appearance / Log out) and an Appearance page listing each `ThemeRow` (28dp color dot + checkmark on the active theme); swiping down on the Appearance page returns to the menu rather than closing the sheet.
- **`App.kt`**: `MaterialTheme(colorScheme = appTheme.toColorScheme())` replaced the hardcoded `EmberColorScheme` reference.

## Consequences
**Positive**
- Reuses [[ADR-002 - Material 3 Dark Default]]'s `EveTheme`/M3 wrapper unchanged — every theme is still a plain `darkColorScheme`, so no screen needed to change how it reads colors.
- Persisted via the same `SecureStorage` abstraction already used for auth tokens — no new storage mechanism introduced.

**Negative**
- Five hand-picked palettes are a fixed set, not a general theming system — adding a sixth faction/variant means another hardcoded `AppTheme` entry, not a config-driven addition.
- No light-theme variant of any of the five — [[ADR-002 - Material 3 Dark Default]]'s "future theme toggling" deferral still applies; this ADR only expands the *dark* palette choices.

## Alternatives considered
- **Single theme only (status quo)** — rejected: cheap personalization win, players already have a strong faction-color association from the base game.
- **User-customizable arbitrary color picker** — rejected as over-engineered for an MVP; five curated, on-brand palettes are safer than letting users produce low-contrast/inaccessible combinations.

## References
- [[ADR-002 - Material 3 Dark Default]]
- [[Guide - App Store Launch Readiness]]
