---
title: ADR-017 — Neon Outline Card and Icon Treatment
type: decision
tags: [adr, architecture, material3, layer-ui, compose-mp]
aliases: [Plasma Conduit]
created: 2026-07-19
updated: 2026-07-19
sources: []
status: active
adr-status: Accepted
---

# ADR-017 — Neon Outline Card and Icon Treatment

## Status
Accepted 2026-07-19.

## Context
The user explored a "sci-fi neon outline" visual direction in chat — first as an isometric-UI reference screenshot, refined to specifically mean thin glowing wireframe lines rather than 3D stacking, then mocked up as 3 Artifact variants in Minmatar/ember colors, then as the winning variant run across all five real [[ADR-013 - Faction Color Themes|faction color schemes]]. The user confirmed the direction and asked to implement it for real.

Two scope questions were resolved directly with the user before implementation:
- **Icons**: `EveIcons` (`ui/icon/EveIcons.kt`) are filled silhouettes converted from real EVE Online icon sprites (`phobiacide/eve-icons`), not stroke art. The lower-risk option raised was keeping the fills and adding a glow halo behind them; the user explicitly chose the higher-fidelity option — redraw as true open-stroke outline art.
- **Nav bar**: `PodNavBar` (`App.kt`) already has a deliberate frosted-glass look (Haze blur pill, spring-animated sliding highlight — see the `dev.chrisbanes.haze` usage in `MainApp`/`PodNavBar`). The user asked to leave that container exactly as built and change only the icon glyphs inside it.

## Decision

**`GlowCard`** (new, `ui/component/GlowCard.kt`) — a drop-in `Card` replacement: a `colorScheme.primary` border, with glow bleeding both **outward** past the card's own bounds (3 filled rounded rects, falling alpha, drawn via `Modifier.drawBehind` *before* the shape `clip`) and **inward**, hugging the border and fading toward the center (3 strokes centered on the border path, each half clipped away by `.clip(shape)` so only the inward-facing half survives). The net effect reads as a neon tube — brightest right at the edge line, soft on both sides of it — not a spotlight anywhere in the middle of the card. Every color is read from `MaterialTheme.colorScheme`, so one component automatically renders correctly across all 5 `AppTheme` schemes with zero theme-specific branching. See "Two real bugs found via visual review" below — the first implementation got both halves of this wrong.

**Glow technique — layered alpha rects, not `Modifier.blur`.** `GradientProgressBar` (`ui/component/SkillProgressBar.kt`) is reused per-row inside `JobsScreen`'s scrolling job list, not just as a single Dashboard hero element as first assumed. Repeated `Modifier.blur` halos in a scrolling list is an avoidable performance and cross-platform-consistency risk on a KMP target. Every glow in this treatment (`GlowCard`, `GradientProgressBar`, the Dashboard avatar ring) is instead 2–3 extra `drawRoundRect`/`drawCircle` calls at decreasing alpha and increasing size — cheap, deterministic, identical on Android and iOS. Text glow (the ISK balance number on `StatCard`) uses the real `TextStyle(shadow = Shadow(...))` API instead, which is not list-repeated here and has no blur-equivalent cost concern.

**Icons — stroke the existing path data, don't redraw it.** `ImageVector.Builder.addPath(...)` accepts `stroke`/`strokeLineWidth`/`strokeLineCap`/`strokeLineJoin` independently of `fill`. All 5 `EveIcons` builders were changed from `fill = Black` to `fill = null, stroke = Black, strokeLineCap = Round, strokeLineJoin = Round` on their *original* path node lists — no new geometry was authored. This fully preserves fidelity to the real EVE-derived iconography (confirmed on-device: the character-sheet hex/pilot mark, stacked skill pages, ringed planet, and factory silhouette all read cleanly as line art at the 36dp nav bar size with no muddiness). Stroke width is a hand-picked constant (`EVE_STROKE_WIDTH = 6f` in the 128-unit viewport icons' own coordinate space; `1.6f` for `settingsIcon`'s 24-unit viewport) calibrated for that nav bar size specifically, since that's the largest and most prominent real usage.

**Discovered: no new `Theme.kt` tokens needed for glow.** Every existing scheme's `onPrimaryContainer` is already, by construction, a bright light-tint of that theme's `primary` (e.g. Ember's `onPrimaryContainer` = `#FFBB94`, a lightened `#FF8040`) — an exact match for the "glow" accent used in the approved mockup, for all 5 themes. `colorScheme.primary` is the structural/border color and `colorScheme.onPrimaryContainer` is the glow/emphasis color everywhere in this treatment; `Theme.kt`'s five `darkColorScheme(...)` definitions were not touched.

**`AppTheme.gainColor`** (new extension, `Theme.kt`) — wallet-gain and job-complete green was hardcoded as `Color(0xFF3FB950)` independently in both `DashboardScreen.kt` and `JobsScreen.kt`, with no theme awareness. Gallente's own primary is emerald (`#30B858`), so a fixed green sits almost on top of that theme's accent. `gainColor` is green everywhere except Gallente, which shifts to amber (`#E0B84A`) — same semantic meaning, no collision. This also consolidated two independent literals into one source, threaded down via each screen's already-collected `ThemeRepository.themeFlow`.

**Scope**: `GlowCard` replaced plain `Card` on Dashboard (`StatCard`, Training, Recent Activity), `PiScreen.PlanetCard`, and `JobsScreen.JobCard` — every screen that used `Card` at all. `SkillsScreen` has no `Card`/`Surface` usage and needed no card change; it still benefits from the `GradientProgressBar` theme-fix if it renders the same hero progress section. **Explicitly not touched**: `PiScreen`'s `PlanetTypeChip`/`StatusChip` and their per-type color constants (`ColorStopped`/`ColorAmber`/`ColorGreen`) — these encode real EVE domain meaning (planet type, extractor/factory status) independent of the app's faction theme and must stay fixed regardless of which `AppTheme` is active. `SecurityStatusBadge`'s high-security-status green (Dashboard) was also left as a hardcoded literal — a different semantic (safety rating, not wallet gain) that wasn't part of the approved scope; it has the same theoretical Gallente-collision risk as the gain color did and is a reasonable small follow-up.

**Nav bar**: no changes to `App.kt`. `PodNavBar`/`PodNavItem` already reference `EveIcons.*` by name and already tint the active icon via `colorScheme.primary` through `animateColorAsState`; once `EveIcons.kt` renders as strokes, the nav bar picked up the new look automatically through its existing mechanism — confirmed on-device, matching the user's "leave the panel, just change the icons" instruction exactly.

## Two real bugs found via visual review (both worth remembering for future glow work)

The first implementation compiled clean, passed lint, and looked plausible in a static code read — both bugs were only visible on an actual device screenshot, and only once the user compared card edges against the ISK number / progress bar glow already on screen.

1. **Stroke vs. Fill silently kills a "fake glow."** The outer bleed rings were originally drawn with `style = Stroke(width = 1.dp.toPx())` — a thin 1dp *outline* at low alpha, which is nearly imperceptible. `GradientProgressBar`'s glow (already correct, and the reference the user pointed at) uses **filled** rounded rects instead — solid area at low alpha reads as a soft blob/halo; a thin stroked line at the same alpha does not. Fix: drop the `style` parameter entirely (defaults to `Fill`) on every "fake glow" draw call, including the Dashboard avatar ring, which had the identical bug. Rule of thumb: layered-alpha fake-glow rings/circles must be filled, never stroked.
2. **A `Brush.radialGradient` glows in the wrong direction for "glow on the edges."** The card's interior background used `Brush.radialGradient(listOf(primary, cardColor))` — brightest in the **center**, fading toward the edges. The mockup's CSS used an `inset` box-shadow, which is the opposite: brightest **at the border**, fading toward the center. A center-out radial gradient is a plausible-looking but backwards translation of an inset shadow, and the bug was easy to miss on small/square elements (the two stat cards, where the center sits right behind the ISK/SP number anyway) but glaring on the wide Training card and tall Recent Activity card, where it showed up as a stray warm blob floating over unrelated content. Fix: replaced the radial gradient with layered strokes centered *on the border path itself*, each half clipped away by the card's own shape clip so only the inward-facing half renders — the Compose equivalent of a CSS inset shadow. Rule of thumb: an "edge glow" needs a gradient anchored at the edge, not at the shape's centroid.

## Consequences

**Positive**
- Zero new `Theme.kt` color tokens — the treatment automatically renders correctly across all 5 existing faction schemes (confirmed on-device for Ember and Gallente specifically, including the `gainColor` swap in both Dashboard's wallet rows and Jobs' status chip/label).
- Icon conversion carries zero art-production risk since it reuses the real, existing EVE-derived path geometry — the higher-fidelity option the user chose turned out cheap to implement once the `addPath(stroke=...)` API was found.
- The layered-alpha glow technique is deliberately blur-free, so it carries no cross-platform rendering-consistency risk and no measured scroll-performance concern — verified on-device scrolling a 3-planet PI list (6 `GradientProgressBar` + 3 `GlowCard` instances on screen at once).
- One reusable `GlowCard` component means future tuning (glow intensity, corner radius, ring count) happens in one place, not per-screen.

**Negative**
- `SecurityStatusBadge`'s green and any other stray hardcoded semantic colors outside the wallet-gain/job-complete scope weren't audited — a known, deliberate boundary, not an oversight, but still a real gap if a future session doesn't know to look for it.
- Icon stroke width is a hand-tuned constant for one specific render size (36dp nav bar), not a scale-aware/responsive value — if `EveIcons` gain a much larger or smaller real usage site later, the stroke width will likely need re-calibrating for that context too.
- Layered alpha rects fall off in visible discrete steps rather than a true Gaussian blur — coarser than the original Artifact mockup's CSS `box-shadow`/`filter: blur()` up close, though confirmed by the user as matching the mockup at normal viewing distance once both the fill-vs-stroke and gradient-direction bugs above were fixed. If a smoother falloff is wanted later, the noted path is real `Modifier.blur` on specific *singular* hero elements only (the Dashboard training bar, the avatar ring) — never on anything reused inside a scrolling list.

## Alternatives considered
- **Keep filled icons, add a glow halo behind them instead of redrawing as strokes** — the lower-risk option raised during planning; rejected by explicit user choice in favor of true outline fidelity, which the `addPath(stroke=...)` discovery made cheap enough to not need the hedge anyway.
- **Rebuild `PodNavBar` as a flat bordered neon panel** (matching the mockup literally) — rejected by explicit user choice; the existing Haze frosted-glass pill and sliding highlight are a real, working design investment that stays as-is.
- **`Modifier.blur`-based glow everywhere** — rejected once `GradientProgressBar`'s reuse inside `JobsScreen`'s scrolling list was confirmed (it isn't just a single Dashboard hero instance); layered alpha strokes chosen instead as a cheap, deterministic, cross-platform-safe substitute.
- **New `Theme.kt` tokens for a dedicated "glow" color per scheme** — unnecessary once `colorScheme.onPrimaryContainer` was confirmed to already match the approved mockup's glow colors exactly, for all 5 existing schemes.

## References
- [[ADR-013 - Faction Color Themes]] — the `AppTheme`/`ThemeRepository`/color-scheme system this treatment renders through unchanged.
- [[ADR-002 - Material 3 Dark Default]]
- [[Screen - Dashboard]], [[Screen - PI]], [[Screen - Jobs]]
