---
title: ADR-027 - AMOLED Default and Contrast Softening
type: decision
tags: [adr, android, ios, theming, material3, accessibility, amoled]
aliases: [ADR-027, Contrast Softening, softenContrast]
created: 2026-09-21
updated: 2026-09-21
sources: []
status: active
adr-status: Accepted
---

# ADR-027 — AMOLED Default and Contrast Softening

## Status
Accepted 2026-09-21.

## Context
Closed-testing feedback (12 testers, per the Play Console production-access process — see [[Guide - App Store Launch Readiness]]): the app's default appearance reads as "too bright." Ember (Minmatar), the original default per [[ADR-002 - Material 3 Dark Default]], pairs near-white text (`EmOnBg #F0E8E0`) against a dark-but-not-black background (`EmBg #0E0908`) — a measured contrast ratio around 16:1, roughly 3-4x higher than WCAG AA's 4.5:1 minimum for normal text. Every one of this app's 5 themes ([[ADR-013 - Faction Color Themes]]) shares that same "near-white on near-black" shape, so switching the default theme alone would not have addressed the complaint — the text/background contrast itself needed softening.

## Decision
Two changes, both in `composeApp/.../ui/theme/`:

**1. Default theme: AMOLED, not Ember (** (`ThemeRepository.load()`, both the no-stored-preference and the malformed-stored-value fallback branches). AMOLED's `background = #000000` is true black — the theme's whole reason to exist is OLED battery savings ([[ADR-013 - Faction Color Themes]]) — so it stays untouched; only the *foreground* side changes (below).

**2. A single tunable `ColorScheme.softenContrast(amount: Float)` function** (`Theme.kt`), applied once in `AppTheme.toColorScheme()`, rather than hand-picking new hex constants for all ~45 "on-*" roles across 5 themes:
```kotlin
private fun Color.blendToward(target: Color, amount: Float): Color = Color(
    red = red + (target.red - red) * amount,
    green = green + (target.green - green) * amount,
    blue = blue + (target.blue - blue) * amount,
    alpha = alpha,
)

private fun ColorScheme.softenContrast(amount: Float = 0.12f): ColorScheme = copy(
    onPrimary = onPrimary.blendToward(primary, amount),
    onPrimaryContainer = onPrimaryContainer.blendToward(primaryContainer, amount),
    // ...same pattern for onSecondary(Container), onTertiary(Container),
    // onBackground, onSurface, onSurfaceVariant
)
```
Every "on-*" (text/icon) role is blended toward its own base role by one shared `amount`. **Deliberately excluded**: `background`/`surface`/etc. (the dark side — AMOLED's `#000000` must stay exactly that), and `error`/`onError` (a semantic "pay attention" signal, not part of the everyday-brightness complaint — softening it would make error text harder to read, the opposite of the goal).

**`amount = 0.12`, not the originally-requested `0.20`.** Measured (not eyeballed) via WCAG's actual relative-luminance contrast formula before picking a final value:

| amount | Caldari `onSurfaceVariant` | AMOLED `onSurfaceVariant` | Gallente `onSurfaceVariant` |
|---|---|---|---|
| 0.20 | 4.13:1 — **below AA** | 4.26:1 — **below AA** | 4.63:1 OK |
| 0.15 | 4.49:1 — borderline | 4.70:1 OK | 5.06:1 OK |
| 0.12 | 4.72:1 OK | 4.96:1 OK | 5.31:1 OK |

`onSurfaceVariant` is the tightest-margin role in every theme (it starts closer to its background than `onBackground`/`onSurface` do) and is used for real small-text UI (`labelMedium`/`labelSmall`/`bodySmall` — stat labels, timestamps), where WCAG AA's 4.5:1 threshold applies (not the 3:1 large-text exception). At the requested 0.20, two of five themes — including AMOLED, the theme this ADR makes the default — would ship with secondary text technically below the accessibility floor for normal text. 0.12 clears every theme's worst case with real margin while still reading as clearly softer on-device (device-verified, not just computed).

## Consequences

**Positive**
- One number (`amount`) tunes contrast across all 5 themes at once — reproducible, and trivially adjustable again later, unlike 45 independently-chosen hex constants.
- [[ADR-017 - Neon Outline Card and Icon Treatment]]'s glow effect (`GlowCard`, which reads `onPrimaryContainer` as its glow color) automatically softens along with everything else — no separate fix needed, and this was itself likely contributing to the "too bright" complaint given the glow is deliberately vivid.
- Device-verified: fresh install shows AMOLED (true black, blue accent) by default; card text/glow visibly softer than the pre-change Ember default, no visual regressions.

**Negative / watch items**
- All 14 `@Preview` composables across the app (`DashboardPreviewSuccess`, `GlowCardPreviewEmber`, etc.) construct `MaterialTheme(EmberColorScheme)` — the **raw**, pre-`toColorScheme()` object — directly, bypassing `softenContrast()` entirely. Android Studio's Preview panel will keep showing full, un-softened contrast; only the real running app (device/emulator) reflects this change. Cosmetic, IDE-tooling-only, zero real-user impact — not fixed here; flagged as a known small inconsistency if anyone goes looking for why Preview doesn't match the device.
- `amount` is a single global knob — if a future theme's worst-case role turns out tighter than Caldari's/AMOLED's current worst case, 0.12 might need re-measuring rather than being assumed to still be safe. The table above is the reference to redo that check against.

## Alternatives considered
- **Hand-picking ~45 new hex constants directly in each `XColorScheme` definition** — rejected: error-prone to compute by hand, harder to re-tune, and duplicates work across 5 nearly-identical structures for what is fundamentally one adjustment.
- **20% as requested, accepting the AA dip** — rejected once measured; a formal accessibility regression on the *new default theme's* secondary text wasn't worth the difference between 12% and 20%, especially heading into a wider public (post-closed-testing) audience.
- **Softening only AMOLED** (since it's the new default and the true-black background is the most likely source of "too bright") — considered, but the user explicitly asked for all 5 themes; kept for consistency across the theme switcher rather than having AMOLED look different in kind from the other four.

## References
- [[ADR-013 - Faction Color Themes]] — the 5 color schemes this modifies uniformly.
- [[ADR-002 - Material 3 Dark Default]] — original dark-theme decision this refines, not reverses.
- [[ADR-017 - Neon Outline Card and Icon Treatment]] — `GlowCard`'s glow color is downstream of this change automatically.
- [[Guide - App Store Launch Readiness]] — the closed-testing feedback that prompted this.
