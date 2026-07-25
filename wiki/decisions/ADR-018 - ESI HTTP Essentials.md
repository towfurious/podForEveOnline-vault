---
title: ADR-018 - ESI HTTP Essentials
type: decision
tags: [adr, ktor, esi, rate-limit, swr-cache]
aliases: [ESI Etiquette Layer, EsiRateLimitPlugin]
created: 2026-07-21
updated: 2026-07-22
sources: []
status: active
adr-status: Accepted
---

# ADR-018 — ESI HTTP Essentials (User-Agent, cache compliance, rate-limit backoff)

## Status
Accepted 2026-07-21.

## Context
[[Guide - App Store Launch Readiness]] flagged three related P1 gaps, all confirmed absent by direct grep before this work: no `User-Agent` header identifying the app to CCP, no handling of ESI's per-client error budget (`X-Esi-Error-Limit-Remain`/`-Reset`, HTTP 420 "Enhance Your Calm"), and no conditional-request support respecting `Expires`/`ETag` — [[Stale-While-Revalidate Cache]] already documents *serving* stale data correctly, but the underlying Ktor calls always issued unconditional `GET`s, which ESI's own docs call "cache circumvention." The user took P0 items (signing, store forms) and asked what could be done from P1 without further input; these three are pure code with no external dependency (contact address resolved to the public GitHub repo URL, already the [[SecureStorage|privacy policy]]'s contact channel too).

Both the `"sso"`-named and `"esi"`-named Ktor clients (see `PlatformModule`) make calls to `esi.evetech.net` — the `"sso"` client doubles as the public/no-auth client (`publicClient = get(ssoClient)` in `SharedModule.kt`) in addition to its OAuth token-exchange role against `login.eveonline.com`. Any fix had to account for that dual role.

## Decision
A single `HttpClientConfig<*>.installEsiEssentials()` extension (`shared/.../data/remote/http/EsiHttpEssentials.kt`), installed identically on both named clients in both `PlatformModule.android.kt` and `PlatformModule.ios.kt`:

1. **`UserAgent` plugin** — `"PodForEve/0.1.0 (viktor.shavarin.dev@gmail.com)"` (corrected 2026-07-22, was the repo URL — see addendum below). Bump the version segment by hand alongside `androidApp`'s `versionName`; not worth expect/actual-plumbing `BuildConfig`/`Info.plist` version strings into `shared` for one string.
2. **Ktor's own `HttpCache` plugin**, not a hand-rolled ETag column in SQLDelight. It already implements RFC 7234 semantics — honors `Cache-Control`/`Expires`, attaches `If-None-Match`/`If-Modified-Since` on stale entries, and handles `304` transparently — for free, at the HTTP-call level, fully orthogonal to [[Stale-While-Revalidate Cache]]'s app-level SWR (which governs when repositories *decide* to refetch; this governs whether a given refetch actually crosses the network once it decides to).
3. **A custom `EsiRateLimitPlugin`** (Ktor `createClientPlugin` + `on(Send)`) tracking the error-budget headers in a shared `EsiErrorBudget` object: below a low-budget threshold (or on an actual 420), further requests short-circuit locally with `EsiErrorBudgetExhaustedException` instead of spending the budget to zero. [[EsiErrorMapper|util.EsiErrorMapper]] (see `shared/.../util/EsiErrorMapper.kt`) maps that exception type to "ESI is rate-limited right now. Try again shortly." — checked before the auth/timeout/network categories since the exception *type* drives it, not message content.

**Critical scoping detail, found during this work's own selfcheck re-review, not before**: the rate-limit gate only engages when `request.url.host == "esi.evetech.net"`. The first version gated on *any* request through the plugin-installed client — which meant a real ESI 420 (triggered by public-endpoint calls through the `"sso"` client) would also block that same client's *unrelated* OAuth token-refresh POSTs to `login.eveonline.com`, turning a public-endpoint rate limit into a bogus "login failed" error. Fixed before this ever reached a device.

## Consequences

**Positive**
- Removes three confirmed P1 gaps with zero new runtime dependencies (both plugins ship inside `ktor-client-core`, already on the classpath).
- `HttpCache` being host-agnostic and call-level means every current and future ESI GET gets compliant caching automatically — no per-repository wiring, no schema migration.
- The error budget is a single `object` (not per-`HttpClient`-instance state) because the budget itself is CCP-side, shared across both named clients.

**Negative / watch items**
- `HttpCache`'s store is in-memory and process-lifetime — it is *not* cleared by [[ADR-018 - ESI HTTP Essentials|this ADR]]'s own logout changes (see the paired cache-wipe work logged 2026-07-21 in `log.md`), unlike the app-level SQLDelight cache. Low risk in a single-character MVP (every cacheable ESI URL embeds `characterId` in its path, so a different character naturally gets a different cache key), but worth remembering if multi-character support is ever added.
- The rate-limit plugin has no automated test exercising the actual Ktor `Send` pipeline (would need a `MockEngine` harness) — only `EsiErrorBudget`'s pure logic and `EsiErrorMapper`'s handling of the exception type are unit-tested. The host-scoping bug above shipped and was caught by manual re-review, not a test; a `MockEngine`-based regression test is a reasonable follow-up if this area sees more changes.

**Addendum (2026-07-22) — contact channel was wrong, fixed**: this ADR's Context originally justified the repo-URL contact as "already the privacy policy's contact channel too, avoids exposing personal email" — but the repo (`github.com/towfurious/podForEveOnline`) is **private** (confirmed via an unauthenticated `curl` to the GitHub API returning 404, once the user mentioned the project would stay private going forward). A link to it is not reachable by CCP's ESI ops team, App/Play reviewers, or end users at all — a dead contact channel, not a privacy-preserving one. Fixed: both the `UserAgent` string and `PRIVACY_POLICY.md`'s Contact section now use `viktor.shavarin.dev@gmail.com` — the same dedicated non-personal "work" Google account created around the same time for Play Console, so this doesn't reintroduce the original personal-email concern either. The user's own reasoning for keeping the repo private (documented via git history, shareable with a reviewer on request if a store dispute ever needs it) doesn't require public reachability, so no conflict with that decision.

## Alternatives considered
- **Hand-rolled ETag/Expires columns in SQLDelight**: rejected — Ktor's `HttpCache` already implements the exact semantics correctly; adding a bespoke cache table would duplicate framework guarantees for no benefit, against this project's own "trust framework guarantees" convention.
- **`HttpResponseValidator`/`expectSuccess` tuning for 420 detection** instead of a custom plugin: rejected — 420 handling needed to *read response headers and mutate shared state* (the budget), which is naturally a `Send`-phase concern, not a validation-phase one; a custom plugin also allows the pre-emptive short-circuit (not sending a doomed request at all) that a validator alone can't provide.
- **Gate the rate-limit plugin on the `"esi"` client only, skip the `"sso"` client entirely**: rejected — the `"sso"` client's public-endpoint calls (universe type names, corp info, planet names) draw from the same CCP error budget and needed the same protection; the real fix was host-scoping the gate, not narrowing which client gets it.

## References
- [[Guide - App Store Launch Readiness]] — P1 #1 (User-Agent), #2 (rate-limit), #3 (cache compliance).
- [[Stale-While-Revalidate Cache]] — the app-level cache discipline this sits underneath.
- [[ESI Scopes MVP]] — the endpoints this protects.
- [[ADR-019 - Offline Detection]] — the paired P1 item done in the same pass.
