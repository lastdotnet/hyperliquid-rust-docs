---
id: claim.begin_block_ordering_surface
title: Begin-block ordering is partly confirmed, but BOLE and delayed-action placement still need closure
subsystem: execution
promotion: confirmed
status: open
confidence: inferred
scope: local_impl
sources: ["source.abci_state_machine_note", "source.block_lifecycle_page", "source.liquidation_page"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The repo has a concrete begin-block effect surface, but the exact BOLE
placement and delayed-action drain placement are still live truth-maintenance
issues. Local code, working notes, and curated docs should be kept in one
explicit sync loop until that ordering is fully closed.

## Evidence

- [`docs/obsidian/ABCI State Machine.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/ABCI%20State%20Machine.md) carries the widened begin-block effect surface and now explicitly marks delayed-action placement as open.
- [`docs/block-lifecycle/index.html`](/Users/androolloyd/Development/hyperliquid-rust/docs/block-lifecycle/index.html) calls out the local/protocol split and highlights BOLE ordering as an open closure item.
- [`docs/liquidation/index.html`](/Users/androolloyd/Development/hyperliquid-rust/docs/liquidation/index.html) documents BOLE as a top-of-block lane.

## Open Questions

- Whether BOLE belongs in the binary-confirmed 9-effect ordering or in a newer widened local pipeline.
- Whether BOLE should execute before same-block rescue/cancel surfaces for full parity.
- Whether delayed CoreWriter actions drain inside the 9-effect begin-block body or immediately after it in the surrounding execution-state wrapper.

## Implementation Impact

- Keep BOLE in one explicit begin-block hook.
- Treat ordering changes here as claim-level sync events, not local comment edits only.
