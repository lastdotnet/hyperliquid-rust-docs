---
id: claim.resphash_backend_split
title: RespHash backends are chain-scoped, but mainnet serialization is not fully wired
subsystem: hashing
promotion: confirmed
status: open
confidence: confirmed
scope: local_impl
sources: ["source.lthash_serialization_note", "source.app_hash_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The repo explicitly tracks a `Mainnet` versus `Testnet` RespHash backend split.
That split is real, and several formerly-open serializer facts are now closed:
successful G-family actions hash like H-family status responses, and fills use
a separate payload from ordinary ledger updates. The remaining gap is exact
end-to-end local wiring, especially for mainnet-specific families and fill
economic scalars.

## Evidence

- [`crates/hl-engine/src/state_hash.rs`](/Users/androolloyd/Development/hyperliquid-rust/crates/hl-engine/src/state_hash.rs) defines `RespHashMode`, backend-aware field-order tables, and tests that make the current limitation explicit.
- [`docs/obsidian/LtHash Serialization.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/LtHash%20Serialization.md) documents the current mainnet/testnet backend split.
- [`docs/obsidian/App Hash.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/App%20Hash.md) is still part of the response-hash truth surface and must stay consistent with the split.

## Open Questions

- Exact mainnet serializer field orders and action-to-response routing for the remaining response families.
- Which backend differences are pure field-order/layout changes versus broader dispatch-path differences.
- Exact local value derivation for the still-open fill-specific fields.

## Implementation Impact

- Do not treat `RespHashMode::Mainnet` as proof that mainnet-specific hashing is fully implemented.
- Keep replay and docs explicit about which serializer facts are closed versus which are still only partially wired locally.
