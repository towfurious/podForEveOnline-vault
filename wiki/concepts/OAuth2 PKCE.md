---
title: OAuth2 PKCE
type: concept
tags: [concept, auth, oauth2, pkce, sso, esi-scopes]
aliases: [PKCE, Proof Key for Code Exchange]
created: 2026-04-24
updated: 2026-07-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# OAuth2 PKCE

## Summary
Proof Key for Code Exchange is an OAuth2 extension (RFC 7636) that makes the authorization-code flow safe for **public clients** — apps that can't keep a client secret, like mobile apps. It replaces the shared secret with a per-session, cryptographically random "verifier" the attacker can't observe.

## Why here
EVE SSO is used as a public OAuth2 provider. The app is a mobile/desktop client that ships in user hands — it cannot hold a client secret. PKCE is the standard mitigation for the token-exchange leg being intercepted. See [[ADR-008 - OAuth2 PKCE via System Browser]].

## Details

### Flow
1. App generates `code_verifier` — 128 bits of entropy via `SecureRandom`, base64url-encoded to ≥43 chars.
2. App derives `code_challenge = base64url(SHA256(code_verifier))`.
3. App opens EVE login URL in the **system browser** (not an in-app WebView) with params:
   - `response_type=code`
   - `client_id=<our eve app id>`
   - `redirect_uri=eveauth-podforeve://callback` (corrected 2026-07-15 — see [[ADR-008 - OAuth2 PKCE via System Browser]] addendum)
   - `scope=<space-joined scopes from ESI Scopes MVP>`
   - `state=<csrf-random>`
   - `code_challenge=<…>`, `code_challenge_method=S256`
4. EVE redirects back to `eveauth-podforeve://callback?code=…&state=…`. App validates `state`.
5. App POSTs to EVE token endpoint with `grant_type=authorization_code`, `code`, `code_verifier` (plus client_id). No client secret.
6. Response: `access_token` (JWT, TTL ≈ 20 min), `refresh_token` (long-lived). Access token stays **in memory only**; refresh token goes to [[SecureStorage]].
7. On access-token expiry → refresh flow using the stored refresh token.

### System browser per platform
- **Android:** Chrome Custom Tabs (`CustomTabsIntent`). Fallback to `Intent.ACTION_VIEW` if no Custom Tabs provider. Respects the user's default browser — preserves their session cookies for better UX.
- **iOS:** `ASWebAuthenticationSession` — OS-managed, session-isolated by default.

### Why system browser and not WebView
WebViews expose the host app to credential capture and can't share the user's existing browser session. EVE SSO policy (and modern OAuth BCP RFC 8252) forbids WebViews for native apps.

### Logout (added 2026-07-21)
`AuthRepository.logout()` clears the refresh token from [[SecureStorage]] **and** wipes the character/skill-queue/planet/industry-job tables in the [[Stale-While-Revalidate Cache|SQLDelight cache]] in one transaction — closing [[Guide - App Store Launch Readiness]]'s P1 #5 (the privacy policy previously said logout left cached character data on-device until uninstall). `skill_type` (universe type-id → name) is deliberately left alone: it's shared reference data, not data about the user's own character. Ktor's in-memory `HttpCache` (see [[ADR-018 - ESI HTTP Essentials]]) is *not* cleared on logout — low risk today since every cacheable URL embeds `characterId`, but worth remembering if multi-character support is ever added.

**What logout does *not* clear (Android, as of 2026-07-24)**: the EVE SSO web session cookie living in Chrome itself. Custom Tabs share Chrome's real cookie jar by design, so before 2026-07-24 a user could tap "Log out" (correctly wiping everything above) and then tap "Login with EVE Online" again and get silently re-authenticated with zero credential prompt — this is standard, well-known Custom Tabs behavior, not specific to this app. Fixed for Android via ephemeral Custom Tabs (`setEphemeralBrowsingEnabled`) — see [[ADR-008 - OAuth2 PKCE via System Browser]]'s 2026-07-24 addendum for the fix and its device-verified fallback behavior. **iOS has the same underlying exposure and is not yet fixed** — see that same addendum for why (the shipped iOS code doesn't even use the documented `ASWebAuthenticationSession` yet).

### Demo Mode (added 2026-07-24)
`AuthState` gained a fifth-branch sibling, `AuthState.Demo` — entered/exited via `AuthRepository.enterDemoMode()`/`exitDemo()`, both plain state flips with no token exchange and no [[SecureStorage]]/SQLDelight access at all. It exists to satisfy Google Play's "Sign in details" store-review requirement without ever sharing a real EVE Online account with a third party — see [[ADR-022 - Demo Mode]] for the full reasoning and the EVECompanion precedent that motivated it.

## Tradeoffs
- **Pros:** no client secret on device, industry-standard, resistant to code-interception attacks, compatible with EVE's OAuth server.
- **Cons:** extra SHA-256 step, requires custom-URL-scheme wiring per platform, still requires `state` validation by us.

## Related
- [[SecureStorage]] — where refresh tokens live.
- [[ESI Scopes MVP]] — list of requested scopes.
- [[ADR-008 - OAuth2 PKCE via System Browser]].

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
- RFC 7636 (PKCE), RFC 8252 (OAuth 2.0 for Native Apps).
