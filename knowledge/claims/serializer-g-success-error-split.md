---
id: claim.serializer_g_success_error_split
title: G-family action hashes use H-shape success and G-shape error payloads
subsystem: hashing
promotion: confirmed
status: closed
confidence: confirmed
scope: testnet_impl
sources: ["source.re_g_success_format_result", "source.lthash_serialization_note", "source.app_hash_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-05
---

## Summary

Actions routed through the G-family response lane do not hash a 3-field map on
success. They switch to the same 1-field `{"status":"success"}` payload used
by the H-family status serializer. The 3-field G payload appears only on the
error path.

## Evidence

- [`re_results/re_g_success_format.json`](/Users/androolloyd/Development/hyperliquid-rust/re_results/re_g_success_format.json) closes the exact success and error shapes from binary analysis plus local source verification.
- [`crates/hl-engine/src/state_hash.rs`](/Users/androolloyd/Development/hyperliquid-rust/crates/hl-engine/src/state_hash.rs) defines `CancelSuccessResponse` and `ActionErrorResponse` and routes them through separate helpers.
- [`docs/obsidian/LtHash Serialization.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/LtHash%20Serialization.md) and [`docs/obsidian/App Hash.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/App%20Hash.md) are part of the repo truth surface for response hashing.

## Open Questions

- Exact field order for the remaining mainnet-specific families that are not yet fully wired locally.

## Implementation Impact

- Do not serialize `{status, success, error}` on successful G-family actions.
- Treat the G/H split as a routing fact, not as a single serializer with optional fields.
