---
id: claim.action_surface_testnet
title: Current widened testnet action surface is 97 variants with 126 sub-types
subsystem: execution
promotion: confirmed
status: active
confidence: confirmed
scope: testnet_impl
sources: ["source.action_types_note", "source.block_lifecycle_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The current widened testnet action surface is tracked as 90 mainnet variants
plus 7 additional testnet variants, for 97 total top-level variants and roughly
126 sub-types once `VoteGlobal`, `PerpDeploy`, `SetGlobal`, and other
sub-variant families are counted.

## Evidence

- [`docs/obsidian/Action Types.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Action%20Types.md) carries the current family map, wire names, and top action shares.
- [`docs/obsidian/Block Lifecycle.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Block%20Lifecycle.md) carries the widened action-category map and the 97-type claim in the current testnet note.
- [`docs/yellowpaper/action-inventory.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/action-inventory.md) is the curated public-facing table for these families.

## Open Questions

- Whether the latest mainnet build still sits at 90 variants exactly or has diverged again.
- Which testnet-only variants should be treated as temporary feature flags versus durable chain-scoped surfaces.

## Implementation Impact

- Keep action-surface claims explicitly chain-scoped.
- Point readers to the action inventory instead of repeating large family tables in every paper page.
