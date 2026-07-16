---
title: OAuth2 PKCE
type: concept
tags: [concept, auth, oauth2, pkce, sso, esi-scopes]
aliases: [PKCE, Proof Key for Code Exchange]
created: 2026-04-24
updated: 2026-07-15
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
