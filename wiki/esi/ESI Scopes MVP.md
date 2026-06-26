---
title: ESI Scopes MVP
type: esi
tags: [esi, esi-scopes, auth, mvp]
aliases: [MVP Scopes, OAuth Scopes]
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# ESI Scopes — MVP

The set of EVE SSO OAuth2 scopes the app requests at login. Any screen that relies on a scope not in this list is out of scope for MVP. Scopes are requested as a space-joined list in the authorization URL (see [[OAuth2 PKCE]] step 3) and appear in the JWT's `scp` claim once granted.

## MVP scopes

| Scope | Entity | Purpose |
|---|---|---|
| `esi-skills.read_skills.v1` | [[Skill Queue]] | Trained SP totals, per-skill level |
| `esi-skills.read_skillqueue.v1` | [[Skill Queue]] | The training queue itself |
| `esi-wallet.read_character_wallets.v1` | [[Character]] | ISK balance shown on [[Screen - Dashboard]] |
| `esi-planets.manage_planets.v1` | [[Planet]] | PI colonies and layouts for [[Screen - PI]] |
| `esi-industry.read_character_jobs.v1` | [[Industry Job]] | Active research and manufacturing jobs for [[Screen - Jobs]] |

## Scope-granting flow
1. Login URL carries `scope=<space-joined>`; user consents on the EVE SSO page.
2. JWT returned contains the granted scopes in the `scp` claim (array of strings). Validate on receipt — a malformed grant where a scope is missing means a screen will 403; fail loud during login, not later.
3. Revocation in the EVE Developer Portal invalidates the refresh token — app must handle refresh failure gracefully (re-prompt login).

## Out-of-MVP scopes (reserve for later)
- `esi-markets.*` — market screen.
- `esi-assets.*` — full inventory.
- `esi-mail.*` — in-app mail.
- `esi-corporations.*` — corporation features.
- `esi-contracts.*` — contracts.

## Per-endpoint pages
Create individual `ESI — <endpoint>` pages in `wiki/esi/` when we actually wire a call in the data-layer plan (each such page documents request/response/caching gotchas for that endpoint).

## Related
- [[OAuth2 PKCE]] — how we obtain these scopes.
- [[ADR-008 - OAuth2 PKCE via System Browser]].
- [[SecureStorage]] — where the token carrying these scopes lives.
