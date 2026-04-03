# Oracle and Mark Price

Complete oracle pipeline from binary RE of `/tmp/target_bin_analysis`.

Source file: `/home/ubuntu/hl/code_Mainnet/l1/src/clearinghouse/oracle/oracle.rs`

## Pipeline Overview

```
External Feeds (Binance, CME, etc.)
        |
        v
  Validator nodes fetch prices
        |
        v
  ValidatorL1VoteAction (per-validator)
        |
        v
  ValidatorL1VoteTracker (3 fields)
    - prune_bucket_guard
    - action_to_tracker (map of ActionTracker)
    - (3rd field unknown)
        |
        v
  ActionTracker (3 fields)
    - active: bool
    - expire_time
    - vote_tracker: VoteTracker
        |
        v
  VoteTracker (7 fields)
    - version
    - round
    - initial_round
    - round_to_val: map
    - order_ntl
    - max_px
    - ema
        |
        v
  Median voting (stake-weighted)
    - validator_to_value
    - median_value (stake-weighted median)
    - update_median_bucket_guard
        |
        v
  SetGlobalAction (4 fields, ALL CONFIRMED)
    - pxs: Vec<Option<f64>>
    - externalPerpPxs: Vec<Option<f64>>
    - usdtUsdcPx: Option<f64>
    - nativePx: Option<f64>  // CONFIRMED: native token (HYPE) price
        |
        v
  Oracle struct (4 fields)
    - pxs: HashMap<u32, f64>
    - external_perp_pxs: HashMap<u32, f64>
    - err_dur_guard: BucketGuard
    - lpxk (abbreviation, likely last_px_kind or similar)
        |
        v
  Clearinghouse (18 fields, includes oracle)
    - coin_to_oracle_px (derived from Oracle.pxs)
    - coin_to_mark_px  (derived from mark price calc)
    - coin_to_external_perp_px (derived from Oracle.external_perp_pxs)
        |
        v
  Mark Price → Funding → Liquidation
```

## 1. Oracle Sources

### SetGlobalAction (4 fields, CONFIRMED 2026-04-02)
The broadcaster submits consensus oracle prices via `SetGlobalAction`:
```rust
struct SetGlobalAction {
    pxs: Vec<Option<String>>,               // oracle prices, indexed by asset
    externalPerpPxs: Vec<Option<String>>,    // external perp prices (Binance, etc.)
    usdtUsdcPx: Option<String>,             // USDT/USDC exchange rate
    nativePx: Option<String>,               // CONFIRMED: native token (HYPE) price
}
```
**CONFIRMED 2026-04-02**: The 4th field is `nativePx` -- the native token (HYPE) price submitted by the broadcaster alongside oracle and external perp prices.

Governance sub-actions (variant index 0..14):
- `registerAsset`, `setOracle`, `insertMarginTable`, `setFeeRecipient`
- `haltTrading`, `setMarginTableIds`, `setOpenInterestCaps`
- `setFundingMultipliers`, `setMarginModes`, `setFeeScale`
- `setGrowthModes`, `setFundingInterestRates`, `disableDex`
- `setPerpAnnotation`

### Oracle struct (4 fields)
```rust
struct Oracle {
    pxs: HashMap<u32, f64>,                 // consensus oracle prices per asset
    external_perp_pxs: HashMap<u32, f64>,   // external perp prices
    err_dur_guard: BucketGuard,             // error duration tracking
    lpxk: OracleKindMap,                    // last price kind map
}
```

### OracleKindMap
Maps assets to their oracle kind. Values seen:
- `Reserved`, `NO_ERROR`, `universe`, `oraclePx`, `disabled`

The `oracle_kind` field on assets distinguishes price source type.

### OraclePxSerde (3 fields)
Serialization format for oracle prices:
```rust
struct OraclePxSerde {
    px: f64,                // the price
    // 2 more fields (likely timestamp, source)
}
```

## 2. Validator L1 Votes

### ValidatorL1VoteAction
Submitted by validators to propose oracle price updates. Listed in the `Action` enum alongside `ValidatorL1Stream`, `SetGlobal`, etc. (80 total action variants).

The binary error messages confirm:
- `"Invalid oracle price asset="` — price validation
- `"Missing price, keeping old oracle price asset="` — fallback on missing data
- `"Unexpected length of prices to set in oracle: prices length="` — length validation
- `"Invalid external perp price for asset="` — external price validation

### ValidatorL1VoteTracker (3 fields)
```rust
struct ValidatorL1VoteTracker {
    prune_bucket_guard: BucketGuard,
    action_to_tracker: HashMap<ActionType, ActionTracker>,
    // 3rd field
}
```

The tracker manages expiry-based voting rounds. Each `ActionTracker` has:
```rust
struct ActionTracker {
    active: bool,
    expire_time: u64,
    vote_tracker: VoteTracker,
}
```

### L1VoteStakes
Tracks validator stake weights for vote aggregation:
```rust
struct L1VoteStakes {
    // Links to CValidatorProfile, c_staking
    // register_time, last_signer_change_time, delegations_blocked
    // jail status
}
```

### ValidatorL1StreamTracker (separate)
Stream-based oracle updates (continuous rather than vote-based):
```rust
struct ValidatorL1StreamTracker {
    // Fields: validator_to_value, median_value, update_median_bucket_guard
}
```

### Median Voting (StreamTracker, 3 fields) -- CONFIRMED stake-weighted median
**CONFIRMED 2026-04-02**: Oracle prices are determined by **stake-weighted median** of validator submissions. This was previously INFERRED from code structure but is now CONFIRMED from binary RE.
```rust
struct StreamTracker {
    validator_to_value: HashMap<Address, ValidatorValue>,
    median_value: f64,                      // CONFIRMED: stake-weighted median
    update_median_bucket_guard: BucketGuard,
}

struct ValidatorValue {
    value: f64,
    // 2nd field (likely timestamp or weight)
}
```

The `median_risk_free_rate` is a separate field on the Exchange (only 1 occurrence found), used for funding interest rate consensus.

### SignatureTracker
Nested within `validatorL1Votes` — tracks vote signatures per validator:
```rust
struct RawSignatureTracker {
    // 3 fields — inner, assert_unique_values, (3rd)
}
```

## 3. Mark Price

### Computation
From the output stream field names:
```
mark_px_inputs → per-asset mark price computation inputs
spot_px_inputs → spot market price inputs
external_perp_px_inputs → external perp price inputs
oracle_pxs → consensus oracle prices
```

Mark price is derived from the order book via impact prices:
```
mark_px = f(oracle_px, impact_bid_px, impact_ask_px, ema)
```

The output response for per-asset data (`PerpAssetCtx`):
```
markPx, funding, openInterest, prevDayPx, dayNtlVlm,
premium, midPx, impactPxs, dayBaseVlm,
circulatingSupply, totalSupply
```

### Coin-to-Price Maps
Three parallel price maps in the exchange state:
```rust
coin_to_mark_px: HashMap<u32, f64>,           // mark prices
coin_to_oracle_px: HashMap<u32, f64>,         // oracle prices
coin_to_external_perp_px: HashMap<u32, f64>,  // external perp prices
```

### HIP-3 Stale Mark Price (CONFIRMED 2026-04-02)
```rust
// Exchange fields:
hip3_stale_mark_px_guard: BucketGuard,   // guards stale mark detection
hip3_no_cross: bool,                      // disable crossing
```
Function: `refresh_hip3_stale_mark_pxs` — called to refresh stale mark prices.
**CONFIRMED**: HIP-3 stale mark price detection uses a **10-second window**. Mark prices older than 10 seconds are considered stale and refreshed during begin_block (effect #5 of 9).
Companion: `prune_book_empty_user_states` — cleanup function.

### VoteTracker Mark Price Fields
```rust
struct VoteTracker {
    version: u32,
    round: u64,
    initial_round: u64,
    round_to_val: HashMap<u64, _>,
    order_ntl: f64,     // order notional for impact
    max_px: f64,        // maximum price
    ema: f64,           // EMA-smoothed price
}
```

## 4. Impact Price

### Configuration
```rust
default_impact_usd: f64,                     // default impact notional (typically 200 USD)
override_impact_usd: HashMap<u32, f64>,      // per-asset override
clamp: bool,                                  // whether to clamp impact price
```

Impact price determines the "fair" mark price by simulating a trade of `impact_usd` notional against the order book. The bid/ask impact prices bracket the mark price.

Output in API: `impactPxs` (array of [bid_impact, ask_impact]).

## 5. Oracle Price Bounds

### Clearinghouse Fields
```rust
struct Clearinghouse {
    oracle: Oracle,
    total_net_deposit: f64,
    total_non_bridge_deposit: f64,
    perform_auto_deleveraging: bool,
    adl_shortfall_remaining: f64,
    bridge2_withdraw_fee: f64,
    daily_exchange_scaled_and_raw_vlms: _,
    halted_assets: HashSet<u32>,
    override_max_signed_distances_from_oracle: HashMap<u32, f64>,
    max_withdraw_leverage: f64,
    last_set_global_time: u64,
    usdc_ntl_scale: UsdcNtlScaleSerde,
    isolated_external: _,
    isolated_oracle: _,
    moh: _,
    // ... (18 total fields)
}
```

### Distance Bounds
- `has_max_oracle_px` — log message when max oracle price bound is hit
- `has_min_oracle_px` — log message when min oracle price bound is hit
- `override_max_signed_distances_from_oracle` — per-asset max distance
- `max_order_distance_from_anchor` — Exchange-level field, limits order placement distance

Error: `": more aggressive than oracle when open interest is at cap."` — rejection when orders exceed oracle distance at OI cap.

## 6. Anchor Price

### Exchange Field
```rust
max_order_distance_from_anchor: f64,   // maximum distance from anchor for orders
```
The "anchor" is typically the oracle price. Orders placed too far from anchor are rejected, especially when open interest is at cap.

Rejection statuses:
- `positionIncreaseAtOpenInterestCapRejected`
- `positionFlipAtOpenInterestCapRejected`
- `tooAggressiveAtOpenInterestCapRejected`
- `openInterestIncreaseRejected`
- `oracleRejected`

## 7. Funding Configuration

### FundingTracker (8 fields)
Lives within the PerpDex. Connected fields:
```
books, funding_tracker, twap_tracker, perp_to_annotation,
VoteTracker, lower_bound, max_leverage, maintenance_deduction,
margin_tiers, MarginTableId, user_to_data
```

### Key Fields
```rust
asset_to_premiums: HashMap<u32, Vec<f64>>,      // CORRECTED: premium sample vectors (not f64)
override_impact_usd: HashMap<u32, f64>,         // per-asset impact override
clamp: bool,
default_impact_usd: f64,
hl_only_perps: HashSet<u32>,                    // HL-exclusive perps
use_binance_formula: bool,                       // Binance-style funding toggle
perp_to_funding_multiplier: HashMap<u32, f64>,  // per-asset multiplier
perp_to_funding_interest_rate: HashMap<u32, f64>, // per-asset interest rate
```

### Base Rates (via SetGlobalAction)
```
perpAddBaseRate: f64,
perpCrossBaseRate: f64,
spotAddBaseRate: f64,
spotCrossBaseRate: f64,
usingBigBlocks: bool,
```

### CumFunding
Per-position cumulative funding:
```rust
struct CumFunding {
    since_open: f64,
    since_change: f64,
    duplicate: f64,
}
```

## 8. PerpAnnotation (5 fields)
Per-asset annotation for perp configuration:
```rust
struct PerpAnnotation {
    // 5 fields — exact names not directly extractable
    // Context: last_settlement_time, deploy_time, n_reserve_deployments_used
    // disabled_state, total_oi_cap2, oi_sz_cap_per_perp
    // max_transfer_ntl2, max_under_initial_margin_transfer_ntl2
    // daily_px_range, max_n_users_with_positions
}
```

Related: `perpConciseAnnotations` — compact annotation format for gossip/concise frames.

### PerpMeta (6 fields)
```rust
struct PerpMeta {
    marginTableIdToMarginTable: _,
    collateralToken: u32,
    collateralIsAlignedQuoteToken: bool,
    collateralTokenName: String,
    // + 2 more
}
```

### PerpAssetInfo (6 fields)
```rust
struct PerpAssetInfo {
    // 6 fields with abbreviated names: cl, ss, cl, ctx, f, trust
    // (clearinghouse, state, context, funding, trust chain)
}
```

### Perp-to Maps
```rust
perp_to_annotation: HashMap<u32, PerpAnnotation>,
perp_to_funding_multiplier: HashMap<u32, f64>,
perp_to_funding_interest_rate: HashMap<u32, f64>,
perp_to_position: HashMap<u32, _>,
```

## 9. HIP-3 Oracle Updater

HIP-3 schemas have their own oracle updater:
```rust
struct Hip3Schema {
    full_name: String,
    oracle_updater: Address,          // address authorized to update oracle
    fee_recipient: Address,
    override_fee_recipient: Option<Address>,
    asset_to_oi_cap: HashMap<u32, f64>,
    sub_deployers: Vec<Address>,
    deployer_fee_scale: f64,
    last_deployer_fee_scale_change_time: u64,
    // ... (9 total fields)
}
```

API fields (camelCase):
```
oracleUpdater, feeRecipient, assetToStreamingOiCap,
subDeployers, lastDeployerFeeScaleChangeTime,
assetToFundingMultiplier, assetToFundingInterestRate,
totalOiCap, oiSzCapPerPerp, maxTransferNtl,
coinToOiCap, totalNetDeposit
```

Output stream: `write-hip3-oracle-updates` — dedicated output for HIP-3 oracle update events.

## 10. Hyperliquidity (2 fields)

The HLP (Hyperliquidity Provider) system interacts with mark prices:
```rust
struct Hyperliquidity {
    // 2 fields
    // Related: hyperliquidity_ensure_orders
    // Points, spot_to_range, ensure_order_bucket_guard
}
```

Exchange fields:
- `cancel_aggressive_orders_at_open_interest_cap_guard`
- `last_hlp_cancel_time`
- `max_hlp_withdraw_fraction_per_day`
- `hlp_start_of_day_account_value`

## 11. Exchange-Level Oracle Fields (57 total Exchange fields)

Key oracle-related fields in the Exchange god object:
```rust
struct Exchange {
    // ... 57 fields total, oracle-related:
    locus: Locus,                        // 14 fields
    validator_l1_vote_tracker: ValidatorL1VoteTracker,
    validator_l1_stream_tracker: ValidatorL1StreamTracker,
    app_hash_vote_tracker: AppHashVoteTracker,
    max_order_distance_from_anchor: f64,
    hip3_stale_mark_px_guard: BucketGuard,
    hip3_no_cross: bool,
    default_hip3_limits: Hip3Limits,
    // ... (see [[Exchange State]])
}
```

## 12. Output Stream Fields

The node outputs these oracle/mark price related fields per block:
```
mark_px_inputs          — mark price computation inputs
spot_px_inputs          — spot price inputs
external_perp_px_inputs — external perp price inputs
oracle_pxs              — consensus oracle prices
hip3_oracle_updates     — HIP-3 oracle update events
```

## Complete Data Flow

```
1. External feeds (Binance, CME, etc.)
   → Validators fetch via HTTP

2. ValidatorL1VoteAction (per validator)
   → Signed vote with prices

3. ValidatorL1VoteTracker
   → ActionTracker per vote type
   → VoteTracker with round-based voting
   → Stake-weighted median computation

4. SetGlobalAction (from broadcaster)
   → pxs[] = consensus oracle prices
   → externalPerpPxs[] = external perp reference
   → usdtUsdcPx = stablecoin rate

5. Oracle struct
   → pxs stored in oracle
   → coin_to_oracle_px derived
   → external_perp_pxs stored
   → coin_to_external_perp_px derived

6. Mark Price calculation
   → impact_bid = simulate buy of impact_usd notional
   → impact_ask = simulate sell of impact_usd notional
   → mark_px = f(oracle_px, impact_bid, impact_ask, ema)
   → coin_to_mark_px stored

7. Funding rate
   → premium = (mark_px - oracle_px) / oracle_px
   → EMA smoothing (DeterministicEma)
   → Apply multiplier (perp_to_funding_multiplier)
   → Add interest (perp_to_funding_interest_rate)
   → Apply base rates (perpCrossBaseRate, perpAddBaseRate)
   → Optional: use_binance_formula toggle

8. Order validation
   → Distance from anchor check
   → Oracle bounds check (has_max/has_min)
   → OI cap enforcement
```

## Links
- [[Clearinghouse]] — oracle stored within clearinghouse
- [[Funding]] — funding rates derived from oracle/mark
- [[Exchange State]] — full exchange god object
- [[Governance]] — SetGlobalAction configuration
- [[Matching Engine]] — order book for impact prices
- [[Guards and Limits]] — bucket guards for rate limiting
- [[Gossip Protocol]] — heartbeats contain concise oracle data

#oracle #mark-price #funding #validators #consensus
