---
title: WalletJournalEntry
type: entity
tags: [domain, wallet, character]
aliases: []
created: 2026-07-13
updated: 2026-07-13
sources: [[Source - 2026-04-24 - EVE Online KMP Design Spec]]
status: active
---

# WalletJournalEntry

## Summary
One entry from the ESI wallet journal (`/v6/characters/{id}/wallet/journal/`). Represents a single ISK movement — income (positive `amount`) or expense (negative). Displayed in the Dashboard "Recent activity" card.

## ESI shape
```json
{
  "id": 25796779950,
  "date": "2026-07-11T01:24:39Z",
  "ref_type": "planetary_construction",
  "amount": -900000.0,
  "description": "Planetary Construction: ToWFurious built on Nakugard IV"
}
```
ESI also returns `balance`, `context_id`, `context_id_type`, `first_party_id`, `second_party_id`, `reason` — not captured in the current DTO (not needed for the dashboard widget).

## Business rules / invariants
- `amount` is positive for income (bounties, market sales, rewards) and negative for expenses.
- `description` is a human-readable string provided by ESI — use it as the primary display label; fall back to `displayName` mapping if empty.
- Filtered ref_types (excluded before display — mechanical noise, not meaningful activity):
  - `market_escrow` — escrow releases / buy-order cancellations
  - `planetary_construction` — PI structure build costs (produces 45+ entries for a 5-planet setup, completely burying real events like market sales)
- At most 5 entries are shown on Dashboard (was 3 before 2026-07-13 fix).

## displayName mapping (fallback when description is empty)
| `ref_type` | `displayName` |
|---|---|
| `bounty_prizes` | "Bounty prizes" |
| `market_transaction` (amount ≥ 0) | "Market sale" |
| `market_transaction` (amount < 0) | "Market buy" |
| `planetary_construction` | "PI construction" |
| `transaction_tax` | "Sales tax" |
| `air_career_program_reward` | "AIR reward" |
| `skill_purchase` | "Skill purchase" |
| `contract_price` | "Contract" |
| `player_donation` | "Transfer" |
| `manufacturing` | "Manufacturing fee" |
| `research_fee` | "Research fee" |
| `agent_mission_reward` | "Mission reward" |
| `structure_gate_jump` | "Jump fee" |
| *(else)* | capitalize each word of `ref_type` split by `_` |

## Implementation notes
- **DTO**: `shared/.../data/remote/esi/dto/EsiWalletJournalEntryDto.kt` — `description` field added 2026-07-13.
- **Domain model**: `shared/.../domain/model/WalletJournalEntry.kt`.
- **Repository**: `CharacterRepository.observeWalletJournal()` — filters `JOURNAL_NOISE_TYPES` (`market_escrow`, `planetary_construction`), sorts descending by date, `take(5)`.
- **UI**: `DashboardScreen.WalletJournalRow` — `entry.description.ifEmpty { entry.displayName }` as label, `TextOverflow.Ellipsis`, maxLines=1.

## Related entities
- [[Character]] — journal belongs to a character.
- [[Screen - Dashboard]] — displays the journal entries.
