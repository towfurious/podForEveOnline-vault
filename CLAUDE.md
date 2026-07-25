# CLAUDE.md — EVE Online KMP Project Wiki

> **Schema and operating manual for the LLM Wiki agent maintaining this Obsidian vault.**
> Any Claude Code (or other LLM) session opening this vault MUST read this file first. It defines your role, the vault layout, conventions, and the workflows you follow when ingesting sources, answering queries, linting, and integrating with code work.
>
> **Language rule: all wiki content MUST be written in English.** This applies to page bodies, frontmatter descriptions, table headers, log entries, comments, and version notes. The user may communicate in any language; the wiki is always English.

---

## 1. Role

You are simultaneously:

1. **Developer agent** — you implement the EVE Online KMP + Compose Multiplatform app. Code lives in the parent directory (`..`) outside this vault. Spec: `[[Source - 2026-04-24 - EVE Online KMP Design Spec]]` (once ingested).
2. **LLM Wiki agent** — you maintain *this* Obsidian vault as a persistent, compounding knowledge base for the project. The vault is the project's memory across time.

These roles run **concurrently**. Every dev session is also a wiki session. As you learn, decide, and build, you update the wiki — creating and linking pages so that future sessions can reconstruct the *why* behind the code without reverse-engineering commits.

Your core mandate as wiki agent:
- Read raw sources, integrate them into wiki pages, maintain cross-references.
- Keep `index.md` and `log.md` up to date.
- Flag contradictions, stale claims, orphans, and gaps.
- Be disciplined about the conventions below — the value compounds only if they are followed.

---

## 2. Vault layout

```
vault/                    (this folder)
├── CLAUDE.md             — this schema (you are reading it)
├── README.md             — short human-facing intro
├── index.md              — catalog of all wiki pages (you maintain)
├── log.md                — chronological log of all operations (you append)
├── sources/              — raw sources + summary pages
│   ├── README.md
│   └── raw/              — immutable source files (new sources go here)
├── wiki/                 — your writable knowledge base
│   ├── entities/         — domain entities (Character, Skill Queue, Planet, …)
│   ├── concepts/         — technical / architectural concepts (OAuth2 PKCE, UiState, …)
│   ├── decisions/        — ADRs (numbered, append-only)
│   ├── patterns/         — reusable code patterns / idioms
│   ├── esi/              — ESI endpoint references
│   ├── screens/          — UI screen designs
│   ├── platform/         — Android / iOS specifics
│   └── guides/           — how-tos and runbooks
├── templates/            — Obsidian page templates (copy when creating a page)
└── .gitignore
```

Rules:
- `sources/raw/` is **immutable** — you read, never modify.
- `wiki/`, `index.md`, `log.md` are yours to write and update.
- ADRs in `wiki/decisions/` are **append-only**. Instead of editing an accepted ADR, write a new one that supersedes it and link both ways.

---

## 3. Conventions

### 3.1 Filenames

Obsidian-native: Title Case with spaces, no prefixes or IDs unless listed below.

| Type | Pattern | Example |
|---|---|---|
| Entity / Concept / Pattern / Screen / Platform / Guide | `Title Case.md` | `Skill Queue.md`, `OAuth2 PKCE.md` |
| ADR (decision) | `ADR-NNN - Title.md` | `ADR-001 - KMP Compose Multiplatform.md` |
| ESI endpoint | `ESI - <collection> - <name>.md` | `ESI - Skills - Get Skill Queue.md` |
| Source summary | `Source - YYYY-MM-DD - Title.md` | `Source - 2026-04-24 - EVE Online KMP Design Spec.md` |
| Template | `<Type> Template.md` | `Entity Template.md` |

### 3.2 Frontmatter (YAML)

Every page starts with:

```yaml
---
title: <exact page title, matches filename without extension>
type: entity | concept | decision | pattern | esi | screen | platform | guide | source
tags: [<tag>, <tag>, ...]
aliases: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [[<source page>]]
status: active | draft | stale | superseded
---
```

- `active` is the default for trusted pages.
- `draft` — WIP, don't rely on claims here yet.
- `stale` — flagged by lint; content may be outdated.
- `superseded` — replaced; body MUST link to the successor page.
- Bump `updated:` every time you edit a page non-trivially.

### 3.3 Links

- Use Obsidian wikilinks: `[[Character]]`, `[[Skill Queue]]`, `[[ADR-001 - KMP Compose Multiplatform]]`.
- Display override: `[[character|the player character]]`. Prefer natural prose where the bare link reads well.
- **Every page** links out to ≥1 related page. Orphans are a lint finding.
- When you create a new page, add inbound links from ≥1 existing page that would naturally reference it.

### 3.4 Tags

Tags go in frontmatter YAML list and optionally inline as `#tag` in prose. Taxonomy grows organically — add a tag when it naturally repeats across ≥2 pages.

Starter taxonomy:

- **Domain:** `domain`, `character`, `skill`, `skill-queue`, `planet`, `pi-extractor`, `pi-factory`, `industry-job`, `blueprint`, `structure`, `station`.
- **Tech:** `kmp`, `compose-mp`, `material3`, `ktor`, `sqldelight`, `koin`, `voyager`, `coil3`, `kotlin`.
- **Platform:** `android`, `ios`, `foreground-service`, `alarm-manager`, `bg-app-refresh-task`, `keystore`, `keychain`.
- **Auth:** `auth`, `oauth2`, `pkce`, `sso`, `esi-scopes`, `secure-storage`.
- **Architecture:** `architecture`, `layer-ui`, `layer-domain`, `layer-data`, `layer-platform`, `ui-state`, `swr-cache`, `expect-actual`.
- **Process:** `adr`, `mvp`, `out-of-scope`, `lint-todo`, `source`.

Merge near-duplicates (`#auth` vs `#authentication`) during lint.

### 3.5 Page types and minimum sections

**entity** — a domain entity.
```
# <Name>
## Summary
## ESI shape
## Business rules / invariants
## Lifecycle / state
## Related entities
## Sources
```

**concept** — a technical / architectural concept.
```
# <Concept>
## Summary
## Why here
## Details
## Tradeoffs
## Related
## Sources
```

**decision (ADR)** — architectural decision record. Append-only once `Accepted`.
```
# ADR-NNN — <Title>
## Status
Accepted YYYY-MM-DD  (or) Superseded by [[ADR-XXX]]  (or) Rejected
## Context
## Decision
## Consequences
## Alternatives considered
## References
```

**pattern** — reusable code idiom.
```
# <Pattern>
## When to use
## Shape
## Example
## Anti-patterns
## Related
```

**esi** — ESI endpoint reference.
```
# ESI — <endpoint>
## Path + method
## Scopes required
## Request shape
## Response shape
## Caching (ESI cache headers)
## Gotchas
## Maps to [[<entity>]]
```

**screen** — UI screen design.
```
# Screen — <Name>
## Goal (5-second smoke check)
## Components
## Data it needs
## States: Loading / Success / Error
## Interactions
## Current UI          ← screenshot + version table (see §4.5)
## Implementation notes  (filled during dev — code paths, files)
## Related ViewModels / UseCases
```

**platform** — Android / iOS specifics.

**guide** — step-by-step runbook.

**source** — summary of an ingested raw source.
```
# Source — YYYY-MM-DD — <Title>
## Provenance
## One-page summary
## Key facts / claims
## Open questions
## Wiki pages touched
```

---

## 4. Workflows

### 4.1 Ingest

Trigger: user drops a file into `sources/raw/` and asks to ingest, or material worth archiving appears in conversation.

Steps:
1. Announce: "Ingesting `<filename>`."
2. Read the source fully.
3. Discuss key takeaways with the user (3–5 bullets). Get a signal before integrating.
4. Create `sources/Source - YYYY-MM-DD - <Title>.md` from `templates/Source Template.md`.
5. Integrate: for each fact, entity, decision, ESI detail, create or update the relevant page in `wiki/`. Wikilink **both directions**.
6. Update `index.md` — add new pages under the right section.
7. Append `log.md`:
   ```
   ## [YYYY-MM-DD] ingest | <Title>
   - Pages touched: [[Page A]], [[Page B]], …
   - Key changes: <2–3 bullets>
   ```
8. Report to user: pages created, pages updated, open questions.

A single source typically touches 5–15 wiki pages.

### 4.2 Query

Trigger: user asks a question against the wiki.

Steps:
1. Read `index.md` first to locate candidate pages.
2. Read candidates, follow wikilinks as needed.
3. Answer with inline citations: "`[[Character]]` says …".
4. If the synthesis is valuable (comparison, new connection, analysis), **file it back as a new page** (usually a `concept` or `guide`) and link it from relevant pages. Otherwise knowledge leaks back into chat history and is lost.
5. Append `log.md`:
   ```
   ## [YYYY-MM-DD] query | <question in ≤10 words>
   - Pages read: [[…]]
   - New page: [[…]]  (if any)
   ```

### 4.3 Lint

Trigger: on explicit request, or proactively after ~10 ingests.

Check:
- **Contradictions** — same claim stated differently on two pages.
- **Stale pages** — older pages whose claims a newer source invalidated.
- **Orphans** — pages with zero inbound or outbound wikilinks.
- **Missing pages** — concepts mentioned on ≥2 pages but lacking their own page.
- **Missing cross-refs** — pages that should link but don't.
- **Tag drift** — near-duplicate tags; merge / normalize.
- **Gap suggestions** — web searches or docs that would close a factual gap.

Output:
- Fix unambiguous issues inline.
- For ambiguous calls, produce a checklist for the user.
- Append `log.md`:
   ```
   ## [YYYY-MM-DD] lint | <scope>
   - Fixed: …
   - Proposed: …
   ```

### 4.4 Dev-to-wiki integration (project-specific rule)

The rule that keeps the wiki alive alongside code. **The dev agent MUST update the wiki when it:**

| Code event | Wiki update |
|---|---|
| Picks an approach where alternatives exist | New ADR in `wiki/decisions/` |
| Adds or replaces a library | `concept` page for the library + ADR |
| Discovers a non-obvious ESI shape or quirk | `esi` page + entity page |
| Hits an Android / iOS quirk | `platform` page |
| Ships a screen | `screen` page gains an "Implementation notes" section with code paths |
| UI of a screen changes visibly | Update `## Current UI` on that screen's page (see §4.5) |
| Accepts code-review feedback that changes an approach | ADR or update affected pages |
| Introduces a reusable code idiom | `pattern` page |

Before claiming a dev task complete, ask yourself: *are there wiki updates owed?* If yes, do them, then append `log.md`:
```
## [YYYY-MM-DD] dev | <task>
- Code: <commits / files>
- Wiki: [[pages touched]]
```

### 4.5 UI screenshot versioning

Every screen page has a `## Current UI` section with the latest screenshot and a version history table.

**Folder:** `vault/attachments/` (Obsidian renders images natively via `![[filename]]`).

**Naming convention:** `screen-{slug}-{YYYY-MM-DD}.png`
Examples: `screen-dashboard-2026-07-09.png`, `screen-pi-2026-07-10.png`.

**Section structure:**
```markdown
## Current UI

![[screen-dashboard-2026-07-09.png]]

| Version | Date | Changes |
|---------|------|---------|
| v2 | 2026-07-10 | Added new feature |
| v1 | 2026-07-09 | Initial implementation |
```

**Workflow when UI changes:**
1. User drops a screenshot into chat.
2. Agent saves it to `attachments/screen-{slug}-{YYYY-MM-DD}.png`.
3. Agent updates `## Current UI`: swaps `![[...]]` to the new file, appends a row to the version table (old screenshot stays in folder as archive).
4. Agent appends to `log.md`:
   ```
   ## [YYYY-MM-DD] dev | UI screenshot — Screen - <Name>
   - Added screenshot v{N}: [[screen-{slug}-{YYYY-MM-DD}.png]]
   - Changes: <brief description>
   ```

**Placeholder before screenshot is received:**
```markdown
> _Screenshot pending — save the file to `attachments/` and replace this line with `![[screen-{slug}-YYYY-MM-DD.png]]`._
```

---

## 5. index.md

`index.md` is the catalog. One line per page:

```
- [[Page Title]] — one-line description
```

Sections match directory structure: Entities, Concepts, Decisions, Patterns, ESI, Screens, Platform, Guides, Sources. Plus an **Open threads** section at the bottom listing `status: draft` pages and lint-flagged items.

Update on every ingest, query that produces a new page, or lint pass. Keep it skimmable — if any section grows past ~30 entries, split by subtopic.

---

## 6. log.md

Append-only chronological record. Entry format:

```
## [YYYY-MM-DD] <type> | <short title>
- <bullet>
- <bullet>
```

Types: `ingest`, `query`, `lint`, `dev`, `meta`.

Start each entry on a new line beginning with `## [`. This makes `grep "^## \[" log.md | tail -20` yield the last 20 events.

Never edit past entries. If a past entry was wrong, add a new `meta` entry correcting it and move on.

---

## 7. Templates

`templates/` holds copy-paste starters for each page type. When creating a new page, start from the matching template and fill in. If you change a page structure in §3.5, update the matching template.

---

## 8. Git

This vault is its own git repository (`git@github.com:towfurious/podForEveOnline-vault.git`), a sibling directory of the app repo, not nested inside it. **Commit it separately from the app repo, every time** — see the `ship` skill in `podForEveOnline/.claude/skills/ship/SKILL.md`, which exists specifically because this step was being skipped (an entire multi-day session's worth of wiki edits sat uncommitted here on 2026-07-24 while the app repo got clean commits the whole time).

Commit style inside the vault:
- Small, frequent commits. One commit per logical operation (one ingest = one commit, one lint pass = one commit).
- Message starts with a verb and a short scope:
  - `add: <page>` — new page
  - `update: <page>` — non-trivial edit
  - `link: <a> ↔ <b>` — cross-reference added
  - `ingest: <source>` — full ingest op
  - `lint: <scope>` — lint pass
  - `meta: <what>` — schema or index changes

---

## 9. When in doubt

- Prefer one-line answers over paragraphs. Prose must earn its length.
- Prefer linking over restating. If another page already says it, link, don't duplicate.
- Prefer creating a new small page over growing one large page — pages you can hold in context at once are edited more reliably.
- When information in memory / vault conflicts with freshly observed facts: trust what you observe now, update the vault, note the change in the log.
