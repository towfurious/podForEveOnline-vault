---
title: ADR-008 — OAuth2 PKCE via System Browser
type: decision
tags: [adr, auth, oauth2, pkce, sso, secure-storage]
aliases: []
created: 2026-04-24
updated: 2026-07-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-008 — OAuth2 PKCE via System Browser

## Status
Accepted 2026-04-24.

> **Addendum (2026-07-15)**: the redirect URI documented below was corrected during initial Android/iOS setup — the actual registered and shipped scheme is `eveauth-podforeve://callback`, not `eve-tracker://callback` (the EVE Developer Portal requires an `eveauth`-prefixed custom scheme). This page's original body is left as accepted (ADRs are append-only); [[Platform - Android]], [[Platform - iOS]], and [[OAuth2 PKCE]] have been corrected to the shipped value. The `[[Guide - Registering eve-tracker URI]]` mentioned under Consequences below was never written — see [[Guide - App Store Launch Readiness]] for current open documentation gaps.
>
> **Addendum (2026-07-24) — Android login now deliberately breaks the session-cookie reuse this ADR originally listed as a positive**: user noticed that PodForEve's own `Log out` (which correctly wipes the local refresh token + cache, see [[OAuth2 PKCE]] "Logout") could be silently bypassed — tapping login again just reused Chrome's still-live EVE SSO session cookie with zero credential prompt, because Custom Tabs share Chrome's real cookie jar by design (exactly what line 39 below praised). Confirmed via CCP's documented `/v2/oauth/authorize/` parameters that EVE SSO has no `prompt=login`-style server-side way to force re-auth. Fixed client-side instead: `UrlLauncher.android.kt`'s Custom Tabs launch now uses `CustomTabsIntent.Builder().setEphemeralBrowsingEnabled(...)`, gated on `CustomTabsClient.isEphemeralBrowsingSupported()` with a graceful fallback to the original session-sharing behavior when unsupported (no `androidx.browser` version bump needed — already on 1.10.0, above the 1.9.0-alpha05 minimum). **Device-verified the fallback path is real, not just written**: a temporary debug log confirmed `isEphemeralBrowsingSupported()` returns `false` on the test device despite Chrome 150 (above the documented 136+ minimum) — likely gated behind a server-side Chrome rollout flag rather than a pure version check, so this feature will silently start working on its own as Google rolls it out further, no app change needed.
>
> **Real gap found while touching this file**: the Decision above states iOS uses `ASWebAuthenticationSession` — the actual shipped `UrlLauncher.ios.kt` does not; it uses plain `UIApplication.openURL`, opening full Safari rather than an in-app auth session. This predates this addendum and was not introduced by the Android fix above. It means the exact same session-bypass-on-logout problem this addendum just fixed on Android almost certainly also exists on iOS today (Safari shares cookies the same way Chrome does), and fixing it properly would first need switching to the documented `ASWebAuthenticationSession` API (which has its own `prefersEphemeralWebBrowserSession` property for the same fix) before an equivalent ephemeral-session fix is even possible. Not fixed now — iOS work is deliberately deferred until after the Android launch per the user's 2026-07-22 sequencing decision. Tracked as a real, verified gap, not a guess: see [[Guide - App Store Launch Readiness]] P2.

## Context
The app authenticates against EVE SSO as a public mobile client. It cannot hold a client secret, must survive token interception, and should not reinvent the security wheel. OAuth 2.0 for Native Apps (RFC 8252) mandates the system browser; EVE SSO supports PKCE.

## Decision
Authenticate via [[OAuth2 PKCE]] using the **system browser**:

- **Android** — Chrome Custom Tabs (`CustomTabsIntent`), fallback to `Intent.ACTION_VIEW` when no Custom Tabs provider is installed.
- **iOS** — `ASWebAuthenticationSession`.

Redirect URI: **`eve-tracker://callback`**, registered in the EVE Developer Portal, `AndroidManifest.xml` (as an `<intent-filter>` on the entry activity), and `Info.plist` (`CFBundleURLTypes`).

Token handling:
- **Access token** — JWT, in memory only, TTL ~20 min. Used as `Authorization: Bearer …` by Ktor's auth plugin.
- **Refresh token** — long-lived, stored via [[SecureStorage]] under key `eve.refresh_token`. Refresh flow runs on 401 from ESI or proactive TTL check.
- **PKCE verifier / state** — ephemeral, memory-only, discarded after the code exchange.

## Consequences
**Positive**
- Standards-compliant; aligns with EVE SSO policy and RFC 8252.
- Users log in with their existing EVE session cookie (Custom Tabs / ASWebAuth share the browser session).
- No WebView → no credential-capture surface; no secret in binary.

**Negative**
- Custom-URL-scheme wiring is platform-specific and easy to get wrong — needs a guide page (`[[Guide - Registering eve-tracker URI]]`, to write during auth plan).
- Callback handoff can surface weird states on iOS if the user dismisses the `ASWebAuthenticationSession` — UI must handle cancel.
- No way to silently reuse SSO if the user has revoked refresh — must re-prompt.

## Alternatives considered
- **Embedded WebView.** Rejected: against EVE SSO policy and RFC 8252; security-hostile.
- **OAuth without PKCE.** Rejected: public clients without PKCE are vulnerable to auth-code interception on the loopback / custom-URI path.
- **Device-code flow.** Rejected: poor UX on a phone where the browser is one tap away.

## References
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- [[OAuth2 PKCE]]
- [[SecureStorage]]
- RFC 7636, RFC 8252.
