---
title: ADR-014 — Ktlint + Detekt Linter Setup
type: decision
tags: [adr, architecture, kotlin, layer-ui, lint-todo]
aliases: []
created: 2026-07-10
updated: 2026-07-10
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
adr-status: Accepted
---

# ADR-014 — Ktlint + Detekt Linter Setup

## Status
Accepted 2026-07-10.

> Reconstructed 2026-07-15 from the `log.md` entry recorded at the time — this page was never written up when the setup shipped, only referenced (broken) from `index.md`. Found during the [[Guide - App Store Launch Readiness]] audit.

## Context
The project had no automated formatting or static-analysis check before this point — style drift and lint-worthy issues (see the 2026-07-14 sweep in `log.md`, which found 39 accumulated formatting violations across five files) could only be caught by manual review. A KMP project with both Android and Compose Multiplatform code needs tooling that understands Compose idioms (e.g. PascalCase composable function names), not generic Kotlin style rules.

## Decision
- **Ktlint** (`org.jlleitschuh.gradle.ktlint` 12.1.2, formatter 1.3.1) — formatting enforcement, applied in the root `subprojects {}` block so every module is covered uniformly. Config via `.editorconfig`: `intellij_idea` style, 140-char line limit, `ktlint_standard_function-naming` disabled (Composable functions are PascalCase, not camelCase).
- **Detekt** (`io.gitlab.arturbosch.detekt` 1.23.8) — static analysis, `buildUponDefaultConfig = true`. Config at `config/detekt/detekt.yml`: the `formatting` section is removed entirely (Ktlint already owns formatting, avoiding double-reporting), with Compose-friendly threshold overrides (`LongMethod.threshold=80`, `LongParameterList.functionThreshold=8` + `ignoreAnnotated: ["Composable"]`, `MagicNumber.active=false`, `FunctionNaming.ignoreAnnotated: ["Composable"]`, `MaxLineLength=140`).
- **compose-rules** (`io.nlopez.compose.rules:detekt` 0.4.22) — Compose-specific Detekt rules on top of the two above. The correct Maven Central artifact group is `io.nlopez.compose.rules`; `io.github.mrmans0n` (the GitHub org name) does not exist as a published group and will fail resolution if used by mistake.
- **Baselines**: `composeApp/detekt-baseline.xml` and `shared/detekt-baseline.xml`, generated once via `./gradlew detektBaseline` and committed — only *new* issues introduced after the baseline fail a Detekt run, so initial adoption didn't require fixing every pre-existing finding up front.
- **Generated-code exclusion**: Ktlint's filter excludes any path containing `/build/generated/`, so SQLDelight- and Compose-resource-generated files are never linted.
- Adopting this required two small code changes to satisfy the new rules: `ThemeRepository`'s backing property renamed `_theme` → `_themeFlow` (Detekt's `property-naming` requires a backing property to match its public property's name), and an empty placeholder `shared/iosMain/KoinInit.kt` was deleted (the real init already lived in `composeApp/iosMain/KoinInit.kt`).
- `ktlintFormat` was then run once across the whole codebase — 45 files auto-reformatted (whitespace, trailing commas, import ordering), zero logic changes.

## Consequences
**Positive**
- Three-tool split (Ktlint = formatting, Detekt = static analysis, Android Lint = Android-API-aware checks) keeps each tool's responsibility narrow and avoids duplicate findings between Ktlint and Detekt.
- Baseline-first adoption meant the tooling could be turned on immediately without a large one-time cleanup PR blocking everything else.

**Negative**
- Three separate lint/static-analysis tools (Ktlint, Detekt, Android Lint) now exist in this project, each with its own config file, invocation, and failure mode — a maintainer has to know all three exist and where each is configured. This was a real point of confusion the first time Android Lint's `MissingPermission` check was investigated during the notification feature work and turned out to be a *different* tool from either of these two, with its own pattern-matching quirks — see [[ADR-015 - Unified Completion Notifications]].
- Detekt baselines can silently accumulate suppressions over time if not periodically reviewed and pruned.

## Alternatives considered
- **Ktlint only, no Detekt** — rejected: Ktlint only checks formatting, not code-quality/complexity issues (long methods, magic numbers, etc.).
- **Detekt's own formatting rules instead of Ktlint** — rejected: Ktlint's formatter can auto-fix (`ktlintFormat`), Detekt's formatting wrapper historically lags behind standalone Ktlint releases.

## References
- [[ADR-004 - Ktor SQLDelight Koin Coil3 Stack]]
- [[ADR-015 - Unified Completion Notifications]]
- [[Guide - App Store Launch Readiness]]
