---
title: Character
type: entity
tags: [entity, domain, character, mvp]
aliases: []
created: 2026-04-24
updated: 2026-04-24
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# Character

## Summary
The single EVE Online pilot the app tracks. MVP scope is **one** character — multi-character support is out of scope. Character identity carries through every screen and every ESI call.

## ESI shape
Relevant fields pulled across endpoints:

- `character_id: Long`
- `name: String`
- `corporation_id: Long`
- Portrait via `https://images.evetech.net/characters/{character_id}/portrait` — loaded with [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack|Coil 3]].
- Wallet balance (ISK): `GET /characters/{character_id}/wallet/` → scalar double.

Requires scopes from [[ESI Scopes MVP]] — at minimum `esi-wallet.read_character_wallets.v1` for the wallet call; other per-screen scopes are listed on their respective entity pages.

## Business rules / invariants
- Exactly one active character per install (MVP).
- `character_id` is derived from the OAuth2 JWT `sub` claim during [[OAuth2 PKCE]] login; the app never asks the user for it.
- Portrait URL is deterministic from `character_id`; cache it.

## Lifecycle / state
1. **Unauthenticated** — no character, show login.
2. **Authenticated** — access token in memory, refresh token in [[SecureStorage]]; `character_id` decoded from JWT; ESI calls permitted within granted scopes.
3. **Logout** — refresh token purged from `SecureStorage`, in-memory state cleared.

## Related entities
- [[Skill Queue]] — per-character queue.
- [[Planet]] — per-character PI colonies.
- [[Industry Job]] — per-character research/manufacturing jobs.

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
