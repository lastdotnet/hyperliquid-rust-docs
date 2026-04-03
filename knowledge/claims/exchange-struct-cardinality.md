---
id: claim.exchange_struct_cardinality
title: Exchange struct cardinality is currently 57 fields
subsystem: state
promotion: confirmed
status: open
confidence: confirmed
scope: testnet_impl
sources: ["source.exchange_state_note", "source.exchange_struct_chain_run"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
expected_count: 57
---

## Summary

The current best repo truth is that `Exchange` has **57 fields**, not 56.

This should be treated as the governing cardinality for current local docs and
field-count comments until stronger contrary evidence lands.

## Evidence

- [`docs/obsidian/Exchange State.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Exchange%20State.md) says the binary confirms `struct Exchange with 57 elements`.
- [`docs/generated/re/runs/2026-04-03-struct-chain.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/generated/re/runs/2026-04-03-struct-chain.md) is the current generated struct-chain source from the 2026-04-03 testnet binary session.
- [`crates/hl-engine/src/bootstrap_fast.rs`](/Users/androolloyd/Development/hyperliquid-rust/crates/hl-engine/src/bootstrap_fast.rs) currently documents and parses the Exchange surface as 57 fields.

## Open Questions

- Whether the latest mainnet and testnet binaries still agree on `Exchange` cardinality.
- Whether any lingering `56` references in repo prose are stale local notes or reflect a chain-specific binary revision.

## Implementation Impact

- Do not collapse local docs or bootstrap comments to 56 fields without primary evidence.
- Treat any `56` reference as a sync issue to be resolved through the protocol sync report and claim update flow.
