# HIP-4: Outcome Contracts

Prediction markets / event contracts on HyperCore. Binary instruments that settle between 0 and 1 (or fractionally via `settleFraction`).

## Core Design

- **1x isolated margin only** — no leverage, no liquidations, loss capped at premium
- **Collateral**: configurable per deployment via `collateralToken` in `Hip3SchemaInput` — **any aligned quote token**, NOT restricted to USDH
- **Settlement**: authorized `oracleUpdater` posts final outcome → trading halts → auto-settle
- **Fractional settlement** via `settleFraction` — supports partial resolution, not just binary 0/1
- **Named outcomes** for categorical markets (elections, multi-choice) via `namedOutcome` / `namedOutcomes`
- **Questions** group related outcomes (e.g., "Who wins the election?" with multiple named outcomes)
- **Fallback resolution** via `fallbackOutcome` — default if no explicit settlement

## Price Discovery

Outcomes use a fundamentally different price model than perps:

| | Perps (HIP-3) | Outcomes (HIP-4) |
|---|---|---|
| **Price during trading** | `coin_to_oracle_px` (external feed) + `coin_to_mark_px` (book) | **CLOB mid only** — no external oracle feed |
| **Oracle role** | Continuous price feed (Binance/Pyth) for mark/funding | **Settlement only** — `oracleUpdater` posts final 0/1 result |
| **Mark price** | Derived from oracle + book | Derived from **book only** |
| **Stale mark handling** | `refresh_hip3_stale_mark_pxs` / `hip3_stale_mark_px_guard` | Same infrastructure (shared HIP-3 book system) |

The `oracleUpdater` address is set at deployment time in `Hip3SchemaInput`. It is the **deployer-designated settlement authority** — the deployer IS the oracle. There is no Chainlink, Pyth, or continuous feed for outcome prices.

### Oracle-related fields in the binary
```
oracle_updater      — per-schema settlement authority address
oracle_kind         — oracle type enum (may distinguish outcome vs perp oracle)
oracleUpdater       — serde field name in Hip3SchemaInput
coin_to_oracle_px   — perp oracle prices (NOT used for outcome price discovery)
coin_to_mark_px     — mark prices from orderbook (used by outcomes)
oracle_pxs          — aggregated oracle prices
mark_px_inputs      — inputs to mark price computation
external_perp_px    — external perp prices (perps only)
has_max_oracle_px   — oracle price bounds check
has_min_oracle_px   — oracle price bounds check
```

### Price bounds
- Trading price bounded between **0.001 and 0.999** during active trading
- Settlement at precisely **0 or 1** (or fractional via `settleFraction`)

## Lifecycle

### Phase 1: Deploy (via `Hip3DeployAction` → `OutcomeDeploy`)
- Requires **1M HYPE stake** (slashable for oracle manipulation)
- Deployer defines: collateral token, oracle updater, event schema
- Sub-deployers can be authorized via `Hip3SubDeployers`
- Rate limited: "too many outcomes deployed today"

### Phase 2: Opening Auction (~15 min)
- Single-price clearing auction
- Users submit limit orders, nothing executes during auction
- Engine selects clearing price maximizing matched volume
- All matched orders fill at identical price
- Unmatched orders carry into continuous trading

### Phase 3: Continuous Trading
- Standard CLOB with price-time priority
- Same order infrastructure as perps/spot
- ~200K orders/sec platform-wide
- Supports limit, market, and standard order types
- No leverage — position size = capital at risk

### Phase 4: Settlement
- `oracleUpdater` posts final outcome via `SettleOutcome`
- Trading halts immediately
- Open orders auto-cancelled (`outcomeSettledCanceled`)
- New orders rejected (`outcomeSettledRejected`)
- Positions auto-settle in collateral token at posted outcome
- Optional challenge window before finality

### Phase 5: Slot Recycling
- Resolved slot reuses the 1M HYPE stake for new markets
- Amortizes deployment cost across recurring markets (macro releases, earnings, elections)

## Source File

`/home/ubuntu/hl/code_Mainnet/l1/src/exchange/impl_outcome.rs`

## Token Operations

### SplitOutcome
Lock collateral (in `collateralToken`) and mint both YES and NO tokens. If you deposit 1 unit of collateral, you get 1 YES + 1 NO. The pair always sums to 1 at settlement.

### MergeOutcome
Reverse of split — burn 1 YES + 1 NO to reclaim 1 unit of collateral. This is how you exit without taking a directional view.

### NegateOutcome
Flip a position from YES to NO (or vice versa). Effectively: sell one side, buy the other. Three fields suggest it handles the swap atomically.

### MergeQuestion
Redeem a complete set across all outcomes in a question. For a multi-outcome question (e.g., election with 3 candidates), if you hold tokens for all outcomes, merge reclaims collateral.

## OutcomeDeploy Enum (8 variants, nested inside Hip3DeployAction)

| Variant | Fields | Purpose |
|---------|--------|---------|
| `RegisterOutcomeToken` | 1 | Register backing token for outcome market |
| `RegisterOutcome` | 4 | Create outcome (outcome, settleFraction, details, namedOutcome) |
| `RegisterTokensAndStandaloneOutcome` | 3 | Combined token + outcome registration |
| `RegisterNamedOutcome` | 2 | Add named outcome to existing question |
| `SettleOutcome` | 3 | Oracle posts final resolution |
| `ChangeOutcomeDescription` | 2 | Update outcome metadata |
| `RegisterQuestion` | 3 | Create question grouping multiple outcomes |
| `ChangeQuestionDescription` | 2 | Update question metadata |

## UserOutcomeAction Enum (4 variants)

| Variant | Fields | Purpose |
|---------|--------|---------|
| `SplitOutcome` | 2 (wei, outcome_id) | Lock collateral → mint YES+NO pair |
| `MergeOutcome` | 1 | Burn YES+NO → reclaim collateral |
| `MergeQuestion` | 1 | Redeem complete set across question |
| `NegateOutcome` | 3 | Flip YES↔NO atomically |

## State Structures

### Market Definition
```
OutcomeSpec         — tuple struct defining an outcome
OutcomeSideSpec     — tuple struct for one side (YES or NO)
QuestionSpec        — struct defining a question (group of related outcomes)
OutcomeTracker      — tracks live/active outcomes in Exchange state
```

### Settlement Records
```
SettledOutcome       — resolved state of an outcome
SettledOutcomeSpec   — settlement parameters
SettledOutcomeHistory — historical record of settlements
SettledQuestionSpec  — question-level settlement
```

### Key Fields
- `description` — human-readable event description
- `fallbackOutcome` — default resolution if no explicit settlement
- `namedOutcomes` — named outcomes within a question (categorical markets)
- `settledNamedOutcomes` — already-settled named outcomes
- `settleFraction` — fractional settlement value
- `details` — additional settlement details
- `questions` — list of sub-questions
- `sideSpecs` — specs for each side (YES/NO) of the outcome

## HIP-3 Integration

Outcomes are deployed through the HIP-3 (builder-deployed perps) pipeline. They share infrastructure but have distinct mechanics.

### Shared Infrastructure
- `Hip3SchemaInput` (3 fields): `collateralToken`, `oracleUpdater`, `OutcomeDeploy`
- `Hip3Schema` (9 fields): full_name, oracle_updater, fee_recipient, asset_to_oi_cap, sub_deployers, deployer_fee_scale, last_deployer_fee_scale_change_time, + 2 more
- `Hip3Limits` (6 fields): total_oi_cap, oi_sz_cap_per_perp, max_transfer_ntl, max_under_initial_margin_transfer_ntl, daily_px_range, max_n_users_with_positions
- `Hip3SubDeployers` — delegated deployment authority
- Same margin table infrastructure (`szDecimals`, `marginTableId`, `onlyIsolated`)
- Same orderbook infrastructure (`BookOrders`, `Books`)
- Same `deployer_fee_scale` for fee customization (up to 50% above base)
- `override_fee_recipient` — custom fee destination

### Differences from HIP-3 Perps
- **No leverage** — outcomes enforce `onlyIsolated` with 1x margin
- **No funding** — outcomes don't have funding rates
- **No continuous oracle** — price discovery is purely CLOB-based
- **Settlement** — outcomes have a terminal event; perps are perpetual
- **Token operations** — split/merge/negate are unique to outcomes

## BOLE (Borrow/Lend) Relationship

BOLE is **protocol-controlled, NOT deployer-configurable**:

- **`SystemBole`** (action 46) — broadcaster-only internal operations
  - Sub-variants: Reserve (9), Adl (2), BackstopLiquidatorTokenParams (4), ResetIndex (2), RateLimit (7), TestnetAction (2)
- **`VoteGlobal`** — validators set `Hip3BackstopLiquidatorParams` (min, max, half_life, max_slippage)
- **`BoleAction`** — end-user borrow/lend (operation, token, amount)
- **`BolePool`** (19 fields) — per-token lending pool state

Deployers choose `collateralToken` but **cannot configure lending parameters**. Pool params (reserves, rate limits, ADL, backstop liquidation) are set at protocol level by validator governance.

BOLE state: `totalSupplied`, `totalBorrowed`, `basis`, `value`, `predictedRate`, `ltv`, `portfolioMarginEnabled`, `borrowYearlyRate`, `interestAmount`

Liquidation types: `BoleMarketLiquidation`, `BoleBackstopLiquidation`, `BolePartialLiquidation`

## Order Behavior on Settlement

- `outcomeSettledCanceled` — all open orders auto-cancelled
- `outcomeSettledRejected` — new orders rejected post-settlement
- `hip3CrossMarginReduceOnly` — cross margin positions reduced to close-only

## Info API Endpoints

- `NodeInfoRequest::SettledOutcome` (1 field) — query settled outcomes
- `NodeInfoRequest::OutcomeMeta` — query outcome metadata
- `include_raw_outcome_tokens` — boolean flag to include raw token data
- `outcomePrice` — outcome pricing data
- `GovProposals` — governance proposals (separate system)

## LtHash / State Hash

The "Settlement" category in the 14 total accumulators (11 L1 + 3 EVM) covers outcome settlements:
```
CValidator, CSigner, Na, LiquidatedCross, LiquidatedIsolated, Settlement,
NetChildVaultPositions, SpotDustConversion, BoleMarketLiquidation,
BoleBackstopLiquidation, BolePartialLiquidation
```

## Kalshi Partnership

Hyperliquid and Kalshi announced a partnership (March 2026) to launch on-chain prediction markets together, confirming the collaboration behind HIP-4 is moving toward production.

## Mainnet Rollout

Two-phase approach:
1. **Curated canonical markets** — vetted by the team
2. **Permissionless builder deployment** — anyone with 1M HYPE stake

Currently live on testnet (March 2026).

## Links

- [[Gossip Protocol]] — wire format for consensus
- [[App Hash]] — state hashing includes Settlement category
- [[Exchange State]] — 56-field state includes OutcomeTracker
- [[Governance]] — on-chain governance system (separate from HIP-4)

#hip4 #outcomes #prediction-markets #confirmed
