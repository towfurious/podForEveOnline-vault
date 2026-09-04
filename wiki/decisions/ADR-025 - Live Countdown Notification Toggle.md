---
title: ADR-025 - Live Countdown Notification Toggle
type: decision
tags: [adr, android, notifications, foreground-service, secure-storage, settings]
aliases: [ADR-025, Notification Preferences, Live Countdown Toggle]
created: 2026-09-03
updated: 2026-09-03
sources: []
status: active
adr-status: Accepted
---

# ADR-025 — Live Countdown Notification Toggle

## Status
Accepted 2026-09-03.

## Context
[[ADR-015 - Unified Completion Notifications]] shipped the skill-training live-countdown `ForegroundService` ([[ADR-006 - Android Foreground Service]]) as always-on — no way for a user to opt out of the persistent shade notification, only the completion alert. [[Guide - App Store Launch Readiness]]'s P2 backlog carried this as a named-but-undone item since 2026-07-24: "in-app per-category notification settings toggle." User asked directly whether people should get a choice about tracking progress in the shade.

## Decision
A single boolean preference — **not** a full per-source settings page. Scope deliberately narrow: gates only the *persistent live-countdown* notification, not the "Training complete" completion alert, which stays mandatory (a user who disables the ticker still wants to know when training actually finishes).

- **`NotificationPreferences`** (`shared/commonMain/.../platform/NotificationPreferences.kt`) — same shape as [[ADR-013 - Faction Color Themes]]'s `ThemeRepository`: a `MutableStateFlow<Boolean>` backed by [[SecureStorage]], `skillLiveCountdownEnabled` var as the read/write surface. Defaults to **`true`** (opt-out, not opt-in) — existing installs see zero behavior change until the user touches it. Lives in `shared`, not `composeApp` like `ThemeRepository` — it has to be visible to `NotificationScheduler.android.kt` and `SkillTrainingService`, which `composeApp` doesn't reach.
- **`expect val supportsSkillLiveCountdownNotification`** (`true` Android / `false` iOS) — iOS has no live-countdown concept at all ([[ADR-007 - iOS Local Notifications]], one-shot only), so the settings row is hidden there rather than shown as a no-op toggle.
- **Gate point**: `NotificationScheduler.android.kt`'s `reconcileSkillService()` — the single chokepoint both the live-ESI-fetch path (`SkillQueueRepository.observeSkillQueue()`) and the boot-survival path ([[ADR-020 - Notification Reboot Survival]]'s `BootCompletedReceiver`) already funnel through. One `if (item == null || !notificationPreferences.skillLiveCountdownEnabled)` covers both triggers with no duplicated logic. `reconcileAlarms()` — the backup alarm that fires the completion notification if the service gets killed — runs unconditionally either way, so the completion alert is unaffected by this preference regardless of which path is disabled.
- **Immediate-effect check inside the tick loop**: the gate above only re-evaluates on the *next* `reconcile()` call (next ESI fetch or boot), which could be an arbitrarily long wait if the user just flips the Dashboard toggle mid-training. `SkillTrainingService`'s 60-second tick loop re-checks the preference on every tick and self-stops (`STOP_FOREGROUND_REMOVE`) if it's now disabled, bounding the toggle's real-world latency to ~60s instead of "until the next app foreground/refresh."
- **UI**: new "Live countdown in shade" row in `DashboardScreen.kt`'s existing settings sheet (between Appearance and Log out), first `Switch` in the codebase — added a `SettingsToggleRow` composable alongside the existing plain `SettingsRow`.

## Consequences

**Positive**
- Closes the last of [[ADR-015 - Unified Completion Notifications]]'s three deferred P2 items.
- Reuses every existing chokepoint ([[ADR-015]]'s `reconcile()`, [[ADR-020]]'s boot path) rather than adding a second notification-scheduling code path — same discipline [[ADR-020]] itself followed.
- Device-verified end-to-end on real hardware with a real active skill: toggle off → live notification disappears within the tick interval; toggle on → reappears on the next reconcile; completion alert mechanism untouched in both states.

**Negative / watch items**
- A near-miss during review: an earlier draft tried folding the preference check into the tick loop's `while` condition instead of a second `break`, to avoid a detekt `LoopWithTooManyJumpStatements` finding. That version would have wrongly `REMOVE`d a just-posted completion notification if the toggle happened to be off at the exact moment training completed (a genuine race between two independently-timed events — toggle state and skill completion — that the merged-condition version couldn't distinguish). Reverted to two independent `break`s with a `@Suppress` and a comment explaining why; the two exit paths lead to different cleanup (`REMOVE` vs `DETACH`+`postCompletionNotification`) and shouldn't be merged. Worth remembering if this code is touched again.
- ~60s worst-case latency on toggle-off is a deliberate tradeoff (bounded by the existing tick interval, no new wiring from Composable to Service) rather than a true instant stop — acceptable given this is a low-stakes cosmetic preference, not a safety-critical control.

## Alternatives considered
- **Full per-source notification settings page** (separate toggles for skill/job/extractor completion alerts too) — rejected as scope creep for what the user actually asked; completion alerts across all three sources stay mandatory, matching [[ADR-015]]'s own reasoning that a "did something finish" alert is the actual value proposition, not something people plausibly want off.
- **Immediate stop via a new Composable→Service signal** (broadcast or direct `context.stopService()` call from `DashboardScreen.kt`) instead of the tick-loop self-check — rejected: would need new cross-module plumbing composeApp doesn't otherwise have to platform services, for a UX improvement (60s → instant) not worth the added surface area.
- **Default off (opt-in)** — rejected; every existing install already expects the live countdown (it's been the only behavior since [[ADR-015]]), so opt-out preserves current behavior for everyone who hasn't touched settings.

## References
- [[ADR-015 - Unified Completion Notifications]] — the mechanism this toggle gates; closes its P2 "in-app per-category notification settings toggle" item.
- [[ADR-006 - Android Foreground Service]] — the live-countdown `ForegroundService` itself.
- [[ADR-020 - Notification Reboot Survival]] — the boot-path trigger this gate also covers.
- [[ADR-013 - Faction Color Themes]] — `ThemeRepository`, the pattern `NotificationPreferences` mirrors.
- [[SecureStorage]] — the persistence layer both repositories use.
