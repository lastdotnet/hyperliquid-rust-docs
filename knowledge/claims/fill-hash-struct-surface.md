---
id: claim.fill_hash_struct_surface
title: Fill hashes use a separate 18-field payload and API wrappers add coin and feeToken
subsystem: hashing
promotion: confirmed
status: active
confidence: confirmed
scope: testnet_impl
sources: ["source.re_fill_details_result", "source.re_ledger_serialization_result", "source.lthash_serialization_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-05
---

## Summary

The hashed fill payload is not just a normal `LedgerDelta` variant with API
fields attached. It is a separate 18-field fill struct containing `px`, `sz`,
`startPosition`, `dir`, `closedPnl`, `oid`, `crossed`, `fee`, `builderFee`,
`tid`, `cloid`, `liquidation`, `feeTrialEscrow`, `builder`, `twapId`,
`deployerFee`, `liquidatedUser`, and `markPx`. `coin` and `feeToken` are added
later by API wrapper code and are not members of the hashed fill payload.

## Evidence

- [`re_results/re_fill_details.json`](/Users/androolloyd/Development/hyperliquid-rust/re_results/re_fill_details.json) closes the 18 fill fields plus the 6 `dir` strings.
- [`re_results/re_ledger_serialization.json`](/Users/androolloyd/Development/hyperliquid-rust/re_results/re_ledger_serialization.json) states that fills live on a separate hash path from ordinary ledger updates.
- [`docs/obsidian/LtHash Serialization.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/LtHash%20Serialization.md) is the repo’s deep note for hash-family structure.

## Open Questions

- Exact local derivation of `startPosition`, `closedPnl`, `tid`, liquidation membership, and maker/taker routing for replay parity.

## Implementation Impact

- Keep fill hashing separate from ordinary ledger-update serialization.
- Do not treat API wrapper fields like `coin` and `feeToken` as proof they belong to the hashed fill payload.
