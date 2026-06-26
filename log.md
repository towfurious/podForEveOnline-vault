# Log

> Append-only chronological record. Entry format: `## [YYYY-MM-DD] <type> | <title>`.
> Types: `ingest | query | lint | dev | meta`.

## [2026-04-24] meta | Vault initialized
- Created vault skeleton: `CLAUDE.md`, `README.md`, `index.md`, `log.md`, `templates/`, `sources/raw/`, `wiki/{entities,concepts,decisions,patterns,esi,screens,platform,guides}/`.
- Project: EVE Online KMP + Compose Multiplatform app. Parent repo at `..` (gitignored to exclude this vault).
- Dual-role: dev-agent + LLM Wiki agent. See `[[CLAUDE]]` §1 and §4.4.
- Next proposed operations: (a) ingest spec `../docs/superpowers/specs/2026-04-24-eve-online-kmp-design.md`; (b) write Foundation implementation plan.

## [2026-04-24] ingest | EVE Online KMP Design Spec
- Source raw: `sources/raw/2026-04-24 EVE Online KMP Design Spec.md` (snapshot of approved spec at `~/docs/superpowers/specs/2026-04-24-eve-online-kmp-design.md`).
- Source page: [[Source - 2026-04-24 - EVE Online KMP Design Spec]].
- Pages created (25):
    - Entities: [[Character]], [[Skill Queue]], [[Planet]], [[Industry Job]].
    - Concepts: [[OAuth2 PKCE]], [[UiState]], [[SecureStorage]], [[Stale-While-Revalidate Cache]].
    - Decisions: [[ADR-001 - KMP Compose Multiplatform]], [[ADR-002 - Material 3 Dark Default]], [[ADR-003 - Voyager and Bottom Navigation]], [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]], [[ADR-005 - Math-Based Skill Progress]], [[ADR-006 - Android Foreground Service]], [[ADR-007 - iOS Local Notifications]], [[ADR-008 - OAuth2 PKCE via System Browser]], [[ADR-009 - UiState Sealed Class with Shimmer]].
    - Pattern: [[Math-Based Progress Bar]].
    - Screens: [[Screen - Dashboard]], [[Screen - Skills]], [[Screen - PI]], [[Screen - Jobs]].
    - Platform: [[Platform - Android]], [[Platform - iOS]].
    - ESI: [[ESI Scopes MVP]].
- `[[index]]` rewritten with all sections populated.
- Open threads logged: per-endpoint ESI pages pending, dev-portal registration status, min API / deployment target, SQLDelight versioning strategy, Compose MP iOS M3 audit.
- Key changes: all nine MVP architectural decisions captured as accepted ADRs; all four MVP screens have living pages ready to accept implementation notes during dev.
