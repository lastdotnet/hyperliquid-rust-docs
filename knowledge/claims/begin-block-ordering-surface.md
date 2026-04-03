---
id: claim.begin_block_ordering_surface
title: Begin-block ordering is 9 named effects with BOLE at #3 and ActionDelayer at #8
subsystem: execution
promotion: confirmed
status: closed
confidence: confirmed
scope: testnet_impl
sources: ["source.block_lifecycle_note", "source.abci_state_machine_note", "source.block_lifecycle_page", "source.liquidation_page"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The current governing begin-block order is now closed in-repo: 9 named effects,
with `apply_bole_liquidations` at slot 3 and `update_action_delayer` at slot 8.
Current local `main` matches that ordering.

## Evidence

- [`docs/obsidian/Block Lifecycle.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Block%20Lifecycle.md) gives the fullest current hook map and places delayed CoreWriter execution in effect 8 of the next block's begin-block.
- [`docs/obsidian/ABCI State Machine.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/ABCI%20State%20Machine.md) carries the widened begin-block effect surface and now explicitly marks delayed-action placement as open.
- [`crates/hl-engine/src/exchange.rs`](/Users/androolloyd/Development/hyperliquid-rust/crates/hl-engine/src/exchange.rs) now encodes the same 9-effect ordering in `BEGIN_BLOCK_EFFECT_ORDER`.
- [`docs/liquidation/index.html`](/Users/androolloyd/Development/hyperliquid-rust/docs/liquidation/index.html) documents BOLE as a top-of-block lane.

## Implementation Impact

- Keep BOLE and delayed CoreWriter execution in the named begin-block surface, not as floating post-effects.
- Treat future ordering changes here as claim-level sync events, not local comment edits only.
