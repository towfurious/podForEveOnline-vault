---
title: Guide - Foldable and Large Screen Support
type: guide
tags: [guide, android, ios, compose-mp, material3, adaptive, foldable]
aliases: [Foldable Support, Adaptive Layouts Plan, Large Screen Plan]
created: 2026-11-25
updated: 2026-11-25
sources: []
status: draft
---

# Guide — Foldable and Large Screen Support

## Goal
Make PodForEve work well on Pixel foldables (inner display) and other large windows — tablets, split-screen, desktop windowing — without changing anything on phones. Strategy and phased plan written 2026-11-25 from current Android guidance and an audit of the code. Status `draft` until the user approves the plan; implementation decisions get recorded as an ADR once made.

## Best-practice sources (read 2026-11-25)
- Adaptive app quality guidelines — [overview](https://developer.android.com/docs/quality-guidelines/large-screen-app-quality), [Tier 3](https://developer.android.com/docs/quality-guidelines/adaptive-app-quality/tier-3), [Tier 2](https://developer.android.com/docs/quality-guidelines/adaptive-app-quality/tier-2). These replaced the old "large screen app quality" guidelines.
- [Use window size classes](https://developer.android.com/develop/ui/compose/layouts/adaptive/use-window-size-classes).
- [Support different display sizes](https://developer.android.com/develop/ui/compose/layouts/adaptive/support-different-display-sizes).
- [Make your app fold aware](https://developer.android.com/develop/ui/compose/layouts/adaptive/foldables/make-your-app-fold-aware).
- [Handle configuration changes](https://developer.android.com/guide/topics/resources/runtime-changes).

Key takeaways that shape this plan:
- **Tier 3 "Adaptive ready"** is the Play baseline on large screens. The app fills the window (no letterboxing or compatibility mode) and keeps its state through rotate, fold/unfold and window resize: scroll position, entered text, playback. It must also handle combinations of these changes and work fully in multi-window. **Tier 2** adds adaptive layouts per window size class: a navigation rail instead of a bottom bar, grids whose column count scales, multi-pane where it fits, bottom sheets and buttons that don't stretch full width, and 48dp touch targets. Tier 1 adds posture awareness. The large-screen Play badge requires Tier 2 plus Tier 1.
- **Window size classes**, by width: Compact <600dp, Medium 600–839, Expanded 840–1199, Large 1200–1599, XL ≥1600. They describe the *window*, never the device. Google's test devices: foldable with a 1:1 inner display (**802×802dp, which is Medium**), foldable with a 4:3 inner display (1024×768dp), an 8" tablet, a 10.5" tablet, and a desktop. Medium width is therefore the size that matters most for a Pixel Fold.
- **Decide app-level layout once, at the root**, from the window size class, and pass it down. Content-level composables should use the space they are actually given (`BoxWithConstraints`, adaptive grids), not the window or screen size. Never use `isTablet` or physical screen checks. Provide the same data at every size, so a layout switch never triggers loading.
- **Configuration changes:** keep UI state in `rememberSaveable` and data in ViewModels / ScreenModels; this is required anyway for process death. For Compose-only apps, Google also recommends handling size changes directly (`android:configChanges="screenSize|smallestScreenSize|orientation|screenLayout"`), which gives fluid fold/unfold with no recreation. It explicitly warns against treating that opt-out as the fix for lost state.
- **Fold awareness:** on a Pixel Fold lying flat, the fold doesn't separate the screen (`isSeparating` is false), so a normal two-pane layout needs no hinge handling. Only half-opened tabletop and book postures separate it. That is Tier 1, optional.
- targetSdk 36: on displays with sw ≥600dp the system already ignores orientation and resizability restrictions. PodForEve has none, so it already runs full-screen. It is just unoptimized.

## Current state (audit 2026-11-25)
1. **No window-size awareness anywhere.** Every screen is a single column that stretches to full width on a Fold's inner display.
2. **Bottom navigation is a custom floating pill** with a Haze blur (`App.kt` `PodNavBar`) that overlays the content. Each screen hard-codes its own clearance below the list: `navBottom + 96dp` (Dashboard), `+80` (Skills, Jobs), `+88` (PI). These are four magic numbers for the same thing, and all become wrong once a rail replaces the bar.
3. **Fold and unfold recreate the Activity.** There is no `configChanges` in the manifest. `splashAnimDone` (`App.kt:84`) and the settings-sheet flags (`DashboardScreen.kt:124-125`) use plain `remember`, so the splash replays and an open settings sheet closes. Whether Voyager keeps the selected tab and the ScreenModels (with their data) across recreation is unverified, so Phase 0 measures it.
4. **No `AndroidView` or other interop**, so opting out of size-based recreation is safe (the docs' main caveat doesn't apply).
5. Login buttons stretch full width. The settings `ModalBottomSheet` should already cap at M3's default max width (640dp); to verify.
6. The iOS target includes iPad (`TARGETED_DEVICE_FAMILY = "1,2,7"`). Adaptive code in `commonMain` will apply on iPad too. iOS is deferred, so iPad isn't verified in this pass.
7. **Nothing to test on yet:** the only system image is `android-37.0/google_apis_playstore`, there are no AVDs, and `cmdline-tools` (`avdmanager`) isn't installed.

## Target for this pass
Full **Tier 3**, plus the **Tier 2 UI items** (`Responsive_adaptive_layouts`, `UI_Secondary_Elements`, `Touch_Targets`). Out of scope: Tier 2 keyboard items (the app has no text input) and all of Tier 1 (postures). Ship as **0.2.0**.

## Strategy
1. **One source of truth at the root.** `MainApp()` reads `currentWindowAdaptiveInfoV2().windowSizeClass` from `org.jetbrains.compose.material3.adaptive:adaptive:1.3.0`. It was confirmed on Maven Central that the API lives in `commonMain`, posture is real on Android and a default on iOS, and it depends on foundation/ui 1.10, which CMP 1.11.1 satisfies. From it, `MainApp()` derives a small `WindowLayout` (navigation: `Bar` or `Rail`, plus the bottom clearance content should leave) and provides it once via a CompositionLocal. Voyager tabs take no parameters, so a CompositionLocal is the pragmatic way to "pass it down".
2. **Content layout follows available width, not the device.** Screens decide columns from the width they actually get after the rail (`BoxWithConstraints`, `LazyVerticalGrid(GridCells.Adaptive(...))`). Cards stay width-agnostic, as they already are, filling whatever column they're placed in.
3. **Same data at every size.** All four screens already load everything regardless of size; keep it that way.
4. **State first, then opt out.** Make UI state saveable and hoist list and scroll states above the one-column/two-column branch so position survives the switch where the content is the same list. Only then add the size-based `configChanges` for seamless fold/unfold. Process death and uiMode changes still recreate the Activity, and those must also keep state.
5. **Keep the visual identity.** Build a custom `PodNavRail` in the same pill language as `PodNavBar`, rather than the stock `NavigationSuiteScaffold`, which would replace the current look. The rail sits beside the content rather than over it, so it doesn't need the Haze blur.
6. **Breakpoints:**
   - Compact (<600dp) is exactly today's UI. This covers phones, the Fold's outer screen and narrow split-screen.
   - Medium and wider (≥600dp, including the 1:1 Fold inner at ~802dp) get the rail and two-column content.
   - Grids may reach 3 columns on Expanded and larger widths through `GridCells.Adaptive`.

## Per-screen layout spec (Medium and wider)
| Screen | Compact (unchanged) | Medium+ |
|---|---|---|
| Dashboard | Single scrolling column | Two columns. Left: character header, ISK/SP stats, training card. Right: recent activity. Settings icon stays in the header. |
| Skills | Active skill, then the queue list | Supporting-pane canonical layout. Left (~1/3): active skill, paused banner, total queue time. Right: the queue list. |
| PI | `LazyColumn` of planet cards | `LazyVerticalGrid`, adaptive columns (min ~340dp), same cards |
| Jobs | `LazyColumn` of job cards | Same adaptive grid approach |
| Login | Full-width buttons | Content capped at ~480dp and centered |
| Settings sheet | Bottom sheet | Bottom sheet capped in width (confirm M3's 640dp default applies) |
| Splash | Centered | No change; already centered |

## Phased plan
**Phase 0 — Baseline (no code).**
- Create AVDs for the Google-recommended set: Pixel Fold, Foldable (1:1 inner), Foldable (4:3 inner), Medium Tablet, and optionally the resizable emulator. Without `cmdline-tools` this has to be done in Android Studio's Device Manager, or `cmdline-tools` installed so it can be scripted.
- Run 0.1.1 and record before-screenshots.
- Note what is lost on fold/unfold and rotate: tab, data (skeleton reload or not), splash, scroll position, open sheet.

**Phase 1 — Continuity (Tier 3).**
- `rememberSaveable` for `splashAnimDone` and the settings-sheet flags.
- Fix whatever Phase 0 shows Voyager loses.
- Add `configChanges="screenSize|smallestScreenSize|orientation|screenLayout"` to `MainActivity`.
- Run T-Config_State, T-Config_Combinations and T-Multi-Window_Functionality (split-screen on the Fold and a tablet, resize the divider).
- Done when fold/unfold/rotate/resize on every screen keeps the tab, data, scroll and sheet, and the splash never replays.

**Phase 2 — Adaptive shell.**
- Add the `adaptive` dependency.
- Add `WindowLayout` plus its CompositionLocal in `MainApp()`.
- Add `PodNavRail` with correct insets: start, top, bottom, display cutout in landscape.
- Replace the four hard-coded clearances with the provided value.
- Back handler unchanged.
- Done when Compact is pixel-identical to today and Medium+ shows the rail with no wasted bottom padding.

**Phase 3 — Screen layouts (Tier 2 UI).**
- Implement the per-screen spec above.
- Add `@Preview`s at 412/802/1024dp widths for fast iteration.
- Check touch targets are ≥48dp.

**Phase 4 — Verify and ship.**
- `selfcheck`, then `verify` on every AVD plus a regression pass on the real Pixel 10 Pro XL (Compact).
- New ADR-029 recording the decisions; update the screen pages' `## Current UI`.
- Release 0.2.0.
- Upload large-screen Play screenshots (1:1 and 4:3 inner displays, plus the outer display), as the guidelines recommend for large-screen discoverability.

**Later (Tier 1, not in this pass):**
- Tabletop posture, e.g. Dashboard summary above the fold and the list below.
- Aligning the two-pane split to the hinge in book posture.
- Keyboard navigation and shortcuts.
- Expanded rail with labels at ≥1200dp.
- iPad verification.

## Risks and open questions
- Voyager tab and ScreenModel behavior across recreation is unknown until Phase 0. Worst case is a custom saver for the current tab; data reload is mitigated by the SQLDelight cache anyway (SWR).
- Scroll position across a one-column → two-column switch only maps cleanly when the same list is shown both ways (PI, Jobs, the Skills queue). Dashboard's column split can't keep the exact offset. That is acceptable, since Tier 3 tests the same layout across a configuration change.
- The Haze blur stays on the bar only, because the rail doesn't overlay content.
- Pixel Fold outer display and narrow split-screen are Compact, so they change nothing, which is correct by design.

## Related
- [[ADR-003 - Voyager and Bottom Navigation]] — planned a `NavigationRail` for tablets in MVP scope but never built it. This plan delivers it.
- [[ADR-019 - Offline Detection]] — inset lesson: anything outside a screen's `Scaffold` needs its own `WindowInsets` handling. The same applies to the rail.
- [[ADR-026 - Tab Back-Press Returns to Dashboard]] — the back behavior must be unchanged with the rail.
- [[Guide - App Store Launch Readiness]] — listing screenshots and Play discoverability.
