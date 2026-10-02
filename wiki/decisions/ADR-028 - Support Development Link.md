---
title: ADR-028 - Support Development Link
type: decision
tags: [adr, android, ios, monetization, donations, ccp-license]
aliases: [ADR-028, Buy Me a Coffee link, Support development]
created: 2026-10-02
updated: 2026-10-02
sources: []
status: active
adr-status: Accepted
---

# ADR-028 — Support Development Link

## Status
Accepted 2026-10-02

## Context
Monetization was scoped on 2026-07-17 as "voluntary donations only" (see [[Guide - App Store Launch Readiness]] P2 Monetization). After the first production release went live (2026-10-02, see `log.md`), the user set up a Buy Me a Coffee page. CCP's Developer License Agreement §4.4(b) permits soliciting voluntary donations "solely to offset Developer's costs of maintaining and supporting an Application" provided use of the app is never restricted or conditioned on donating; §7.3 forbids combining CCP marks with other branding. Re-verified against the live license page 2026-09-26.

## Decision
- One settings-sheet row, "☕ Support development", in `DashboardSettingsSheet` (`DashboardScreen.kt`), between the live-countdown toggle and Log out. Tapping it opens `https://buymeacoffee.com/viktor.shavarin` via the existing `rememberUrlLauncher()` (Custom Tab) — no new launching code.
- Label and the Buy Me a Coffee page carry no CCP/EVE branding (§7.3); the page text states support goes to maintaining the app and that every feature stays free (§4.4(b)).
- Nothing in the app is gated, nagged, or conditioned on donating: a single passive row, no prompts, no counters.
- `supportsExternalSupportLink` (`expect val`, `composeApp/.../platform/SupportLink.kt`): `true` on Android, `false` on iOS. Same shape as `supportsSkillLiveCountdownNotification` ([[ADR-025 - Live Countdown Notification Toggle]]).
- Shipped as `versionName 0.1.1`.

## Consequences
- Android users get a visible, optional way to support the app; income is ordinary taxable income (see the tax note in the Guide), not a gift.
- The URL is a hardcoded constant (`SUPPORT_DEVELOPMENT_URL`); changing the page handle needs an app update.
- Google Play's stance on external donation links was previously flagged unverified; Android-first made it the live question. No new permission, SDK, or data collection — the app only opens a URL, so Data Safety and the privacy policy are unaffected.
- **Device-verified 2026-10-02** (Pixel 10 Pro XL, Demo Mode): row renders (emoji included), tap opens the real Buy Me a Coffee page in a Custom Tab, no crash in pid-scoped logcat.

## Alternatives considered
- **Page named after the app**: rejected as unnecessary worry — §4.4(b) explicitly allows donations for the app's upkeep, so a personal-name page is simply the user's preference, not a compliance workaround.
- **Show on iOS too**: deferred — Apple Guideline 3.2.1(vii) excludes external donation links for "Games"; whether this utility is classified as Games is unconfirmed. Fallback would be StoreKit tip IAP.
- **Banner/prompt after N launches**: rejected — nagging risks looking like conditioning use on donation and annoys users.

## References
- [[Guide - App Store Launch Readiness]] (Monetization, Donations compliance bullets)
- [[ADR-025 - Live Countdown Notification Toggle]], [[ADR-008 - OAuth2 PKCE via System Browser]] (the Custom Tab launcher)
