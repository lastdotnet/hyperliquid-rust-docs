---
id: claim.block_lifecycle_phase_map
title: The repo now uses one explicit block lifecycle phase map and hook surface
subsystem: execution
promotion: confirmed
status: open
confidence: confirmed
scope: local_impl
sources: ["source.block_lifecycle_page", "source.abci_state_machine_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The repo now has one explicit block lifecycle reference surface that ties
topology, proposer flow, begin-block, action dispatch, end-block, EVM handoff,
and VoteAppHash into one phase map.

## Evidence

- [`docs/block-lifecycle/index.html`](/Users/androolloyd/Development/hyperliquid-rust/docs/block-lifecycle/index.html) publishes the current lifecycle and hook-point map.
- [`docs/obsidian/ABCI State Machine.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/ABCI%20State%20Machine.md) is still the deeper working note for begin-block and execution sequencing.

## Open Questions

- Which hook points are protocol-invariant versus only local/current implementation.
- Exact closure on begin-block internals where local code and RE notes still diverge.

## Implementation Impact

- New execution changes should be placed against the explicit phase model, not added as ad hoc block-time side effects.
- Future docs should point at this lifecycle surface instead of rewriting the phase map from scratch.
