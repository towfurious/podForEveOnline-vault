---
title: ADR-008 — OAuth2 PKCE via System Browser
type: decision
tags: [adr, auth, oauth2, pkce, sso, secure-storage]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-008 — OAuth2 PKCE via System Browser

## Status
Accepted 2026-04-24.

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
