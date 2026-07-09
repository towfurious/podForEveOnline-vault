---
title: Screen - Splash
type: screen
tags: [screen, splash, animation, layer-ui, compose-mp]
aliases: [Splash, PodSplashScreen]
created: 2026-07-09
updated: 2026-07-09
sources: []
status: active
---

# Screen — Splash

## Goal

Show a branded warp-in animation while auth state resolves. No user action required; transitions automatically to Login or MainApp when auth check completes and animation finishes.

## Components

- Full-screen dark background `0xFF0C0A12`.
- Main pod blob — gradient silhouette of EVE capsule.
- Three mini-pods — same silhouette scaled down, creating a depth/fleet perspective.
- Warp streak — gold line trailing behind each pod during flight.

## States

Single state: animating. No loading / error variants — the screen exists only during the initial auth check which completes in milliseconds.

## Animation sequence

| t (ms) | Event |
|---|---|
| 0 | Main pod begins warp-in from top-right (EaseOutExpo, 380 ms) |
| 0 | Main pod fades in (200 ms) |
| 380 | Mini-pod 1 warp-in (EaseOutExpo, 200 ms) + fade (130 ms) |
| 490 | Mini-pod 2 warp-in |
| 600 | Mini-pod 3 warp-in |
| ~910 | `onFinished()` called → auth gate transitions to next screen |

Warp direction: top-right → final position (COS45 / SIN45 = 45° diagonal). Main pod offset magnitude `sc * 1.15`; mini-pods `sc * 0.40`.

## Implementation notes

**File:** `composeApp/src/commonMain/kotlin/com/podforeve/tracker/ui/component/PodSplashScreen.kt`

### Blob path

188 cubic bezier curves auto-traced from EVE capsule silhouette via Vectorizer.io (1024×1024 SVG space). Normalised to 0..1 in `buildBlobPath(ox, oy, w, h)`.

Centroid: `ox = cx − 0.5002·sc`, `oy = cy − 0.4990·sc`

### Mini-pods

`buildMiniBlob(ox, oy, sc, cx, cy, f)` — calls `buildBlobPath` with `miniSc = sc * f` and offsets the origin so the blob centroid lands at `(ox + cx·sc, oy + cy·sc)`.

Position/scale constants (edit to reposition without risking pivot desync):

```kotlin
private const val M1_CX = …; private const val M1_CY = …; private const val M1_F = 0.068f
private const val M2_CX = …; private const val M2_CY = …; private const val M2_F = 0.140f
private const val M3_CX = …; private const val M3_CY = …; private const val M3_F = 0.118f
```

### Path caching

`SplashPathCache` holds all 4 paths; `rebuildIfNeeded(sc, size)` skips rebuild when `sc` unchanged (scale only changes on first draw or orientation flip). Warp translation applied via `withTransform { translate(dx, dy) }` — zero new Path allocations per frame.

### Gradient

`Brush.linearGradient` bottom-left (gold `F5D060`) → top-right (dark `1C0A30`) across 6 stops.

Rim light: faint warm overlay on left edge via a second `linearGradient` inside `clipPath(blob)`.

## Related

- [[App]] — auth gate that hosts this screen
- `EveIcons` — tab bar icons added in the same session
