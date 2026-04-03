---
id: claim.portfolio_margin_liquidation_mode
title: Portfolio margin liquidates as a generalized cross-margin account
subsystem: liquidation
promotion: confirmed
status: active
confidence: confirmed
scope: protocol
sources: ["source.hyperliquid_portfolio_margin_docs"]
docs_targets: ["whitepaper", "yellowpaper", "status"]
updated: 2026-04-03
---

## Summary

Portfolio margin is not a separate liquidation engine. It is a portfolio-netted
margin mode over eligible assets, with a liquidation trigger defined by the
portfolio margin ratio crossing the protocol threshold.

## Evidence

- Official docs define a dedicated portfolio-margin ratio and liquidation value.
- Official docs state that perps and spot borrows may liquidate first depending
  on oracle update order, so the liquidation order is not a fixed simple queue.

## Open Questions

- Exact binary implementation order for mixed perp and spot-borrow liquidation.
- How much of the PM path reuses the cross-margin request family mechanically.

## Implementation Impact

- Keep PM as an account mode, not a separate universe.
- Route PM liquidation through the liquidation engine with PM-specific account
  summaries and trigger formulas.
