---
id: claim.corewriter_delayed_action_surface
title: CoreWriter re-enters L1 through a delayed action lane with 15 known action types
subsystem: execution
promotion: confirmed
status: active
confidence: confirmed
scope: testnet_impl
sources: ["source.l1_evm_bridge_flow_note", "source.block_lifecycle_note"]
docs_targets: ["whitepaper", "yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

CoreWriter is the EVM-to-L1 bridge for raw actions. The current repo truth is a
precompile entry at `0x3333...3333`, selector `0x17938e13`, with 15 known
action types that enqueue into `ActionDelayer` and later execute through the
named delayed-action lane in the next block's `begin_block`.

## Evidence

- [`docs/obsidian/L1 EVM Bridge Flow.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/L1%20EVM%20Bridge%20Flow.md) documents CoreWriter log processing, raw-action decoding, and queueing.
- [`docs/obsidian/Block Lifecycle.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Block%20Lifecycle.md) ties the delayed CoreWriter lane into the full block phase map.
- [`docs/yellowpaper/index.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/index.md) now summarizes the precompile address, selector, and 15 action-type surface.

## Open Questions

- Exact enable/disable and volatility-mode semantics in the surrounding ActionDelayer control plane.
- Whether the known 15-action list is complete for the current testnet build or only the currently decoded subset.

## Implementation Impact

- Keep CoreWriter treated as a delayed re-entry surface, not a bypass around L1 ordering.
- Route CoreWriter facts through the action-delayer and block-lifecycle claims instead of duplicating competing narratives.
