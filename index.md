# Index

> Catalog of every wiki page. Maintained by the LLM agent per `[[CLAUDE]]` §5.
> One line per page: `- [[Page Title]] — one-line description`.

## Entities
- [[Character]] — the single EVE pilot the app tracks.
- [[Skill Queue]] — ordered list of training skills; source of Skills screen + Dashboard training widget.
- [[Planet]] — a PI colony; status is derived as Active / Needs Attention / Idle.
- [[Industry Job]] — active research or manufacturing job.

## Concepts
- [[OAuth2 PKCE]] — public-client OAuth2 extension we use for EVE SSO.
- [[UiState]] — shared sealed class driving Loading / Success / Error on every screen.
- [[SecureStorage]] — `expect/actual` wrapper over Keystore / Keychain for the refresh token.
- [[Stale-While-Revalidate Cache]] — cache discipline: serve stale instantly, refresh in background.

## Decisions (ADRs)
- [[ADR-001 - KMP Compose Multiplatform]] — core stack choice.
- [[ADR-002 - Material 3 Dark Default]] — theme and zero custom design-system overhead.
- [[ADR-003 - Voyager and Bottom Navigation]] — navigation lib and 4-item NavigationBar with 5th slot reserved.
- [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]] — shared data/infra libraries.
- [[ADR-005 - Math-Based Skill Progress]] — live progress computed on-device, no polling.
- [[ADR-006 - Android Foreground Service]] — persistent shade countdown + AlarmManager.
- [[ADR-007 - iOS Local Notifications]] — UNUserNotificationCenter + BGAppRefreshTask; no FG-service analog.
- [[ADR-008 - OAuth2 PKCE via System Browser]] — Chrome Custom Tabs / ASWebAuthenticationSession.
- [[ADR-009 - UiState Sealed Class with Shimmer]] — UI state contract and Modifier-based shimmer.

## Patterns
- [[Math-Based Progress Bar]] — compute progress from start/end timestamps, tick via coroutine, never poll.

## ESI endpoints
- [[ESI Scopes MVP]] — the 5 OAuth scopes for MVP (per-endpoint pages added when wired in data-layer plan).

## Screens
- [[Screen - Dashboard]] — portrait, ISK, current training, PI/jobs summary.
- [[Screen - Skills]] — big live progress bar + full queue list.
- [[Screen - PI]] — per-planet cards with derived status chip.
- [[Screen - Jobs]] — list of active research / manufacturing jobs with progress.

## Platform
- [[Platform - Android]] — ForegroundService, AlarmManager, Custom Tabs, Keystore.
- [[Platform - iOS]] — UNUserNotificationCenter, BGAppRefreshTask, ASWebAuthenticationSession, Keychain.

## Guides
*(none yet)*

## Sources
- [[Source - 2026-04-24 - EVE Online KMP Design Spec]] — design spec, approved 2026-04-24.

## Open threads
- Per-endpoint ESI pages (list to grow during data-layer plan).
- EVE dev-portal client_id + redirect URI registration status.
- Minimum Android API level + iOS deployment target (not yet decided).
- SQLDelight schema versioning strategy.
- Compose MP iOS maturity audit against our exact M3 components.
