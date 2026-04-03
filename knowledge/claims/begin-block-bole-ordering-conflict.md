---
id: claim.begin_block_bole_ordering_conflict
title: Stale BOLE begin-block ordering conflict is resolved
subsystem: execution
promotion: confirmed
status: closed
confidence: confirmed
scope: testnet_impl
sources: ["source.block_lifecycle_note", "source.abci_state_machine_note", "source.liquidation_page"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

Older local docs had BOLE liquidation at effect 9. The widened block-lifecycle
note and the current local engine both place BOLE at begin-block effect 3, so
the stale conflict is resolved.

## Evidence

- [`docs/obsidian/ABCI State Machine.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/ABCI%20State%20Machine.md) describes BOLE as effect 3.
- [`docs/obsidian/Block Lifecycle.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Block%20Lifecycle.md) confirms the same slot in the widened 9-effect note.
- [`docs/liquidation/index.html`](/Users/androolloyd/Development/hyperliquid-rust/docs/liquidation/index.html) now treats BOLE as the top-of-block lending liquidation lane.

## Implementation Impact

- Do not spread BOLE liquidation hooks across multiple block phases.
- Keep stale effect-9 references out of paper, yellow-paper, and lifecycle pages.
