# BOLE

Hidden lending protocol built into [[Hyperliquid]] L1. Not an EVM contract.

## Discovery
Found via [[Binary Structure]] RE — name "BOLE" appears in governance actions and state structs.

## Types
- **BolePool(19)**: full pool state
- **BoleAction(3)**: user lending actions (operation, token, amount)
- **SystemBoleAction(3)**: system-level BOLE actions

## SetBole Governance Variants
From Ghidra analysis:
- `SetBole::Reserve(9)` — configure reserves
- `SetBole::TestnetAction(2)` — testnet operations
- `SetBole::Adl(2)` — auto-deleveraging for lending
- `SetBole::BackstopLiquidatorTokenParams(4)` — backstop config
- `SetBole::ResetIndex(2)` — reset lending index
- `SetBole::RateLimit(7)` — rate limiting configuration

## User Operations
Via [[Action Types]] `borrowLend` variant:
- `operation` — borrow/supply/repay/defaultLiquidator/optOut
- `token` — which token
- `amount` — how much

The `defaultLiquidator` operation designates a default liquidator for the user's borrow positions.

## Rate Model
- `kink_rate` — interest rate at kink utilization
- `kink_utilization` — utilization threshold
- `min_rate` — minimum interest rate
- Similar to Aave/Compound rate curves

## State
- `BoleToken` — per-token lending state
- `Hip3BackstopLiquidatorParams(3)` — backstop for HIP-3 assets
- Pool state tracks: totalSupplied, totalBorrowed, utilization, etc.

## Error Messages
- "Insufficient collateral for operation"
- "Insufficient balance"
- "Exceeded cap for asset"
- "Only supplying is enabled"
- "Too many operations"
- "borrow_cap > supply_cap"

## Liquidation (CONFIRMED 2026-04-02)
See [[Liquidation Engine]] for complete pipeline.

**CONFIRMED**: Three BOLE liquidation types tracked in LtHash, with **0.8 health factor threshold**:
- **BoleMarketLiquidation** — standard market liquidation of borrow position
- **BoleBackstopLiquidation** — backstop liquidator takes over position
- **BolePartialLiquidation** — partial liquidation (cooldown enforced)

**CONFIRMED details**:
- Health tracked via `healthFactor`, `ltv`, `utilization` on `BoleUserState`
- Liquidation triggered when health factor drops below **0.8** threshold
- Order type `borrowLendLiquidation` (display: "Borrow Liquidation") is used for fills
- `apply_bole_liquidations()` runs in every `begin_block` before trading (effect #3 of 9)

## Links
- [[Exchange State]] — BOLE state in exchange
- [[Action Types]] — borrowLend variant
- [[LtHash]] — BoleBackstopLiquidation, BolePartialLiquidation hash ops
- [[Liquidation Engine]] — complete liquidation pipeline

#lending #defi #bole
