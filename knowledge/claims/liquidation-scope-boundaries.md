---
id: claim.liquidation_scope_boundaries
title: Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes
subsystem: liquidation
promotion: confirmed
status: open
confidence: confirmed
scope: local_impl
sources: ["source.liquidation_page"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The repo now has an explicit liquidation family split:

- perps and portfolio margin share the main liquidation family
- BOLE is a separate begin-block liquidation lane
- spot is not treated as ordinary perp-style liquidation
- outcomes settle through outcome-specific settlement paths, not liquidation

## Evidence

- [`docs/liquidation/index.html`](/Users/androolloyd/Development/hyperliquid-rust/docs/liquidation/index.html) publishes the current product-family split, formula sheet, and ADL/BOLE separation.
- [`crates/hl-engine/src/exchange.rs`](/Users/androolloyd/Development/hyperliquid-rust/crates/hl-engine/src/exchange.rs) and [`crates/hl-engine/src/liquidation.rs`](/Users/androolloyd/Development/hyperliquid-rust/crates/hl-engine/src/liquidation.rs) carry the current local execution split.

## Open Questions

- Exact ADL sizing and ranking formula details.
- Exact portfolio-margin liquidation leg selection order.
- Final BOLE liquidation semantics and penalties under binary-confirmed parity.

## Implementation Impact

- Do not reuse the perp liquidation path for outcomes.
- Do not describe BOLE as just another perp liquidation variant.
- Keep PM as an account mode inside the main liquidation family rather than a separate engine.
