# Liquidation Engine

Complete reverse engineering of the Hyperliquid liquidation pipeline from binary analysis of `/tmp/target_bin_analysis`.

## Source Files (from binary panic paths)

| Source File | Purpose |
|------------|---------|
| `l1/src/clearinghouse/clearinghouse.rs` | Main clearinghouse, liquidation orchestration |
| `l1/src/clearinghouse/cl_trait.rs` | Clearinghouse trait, `check_all_users_for_liquidatables` |
| `l1/src/clearinghouse/user_state.rs` | Per-user margin state, liquidation eligibility |
| `l1/src/clearinghouse/user_states.rs` | Batch user state operations |
| `l1/src/clearinghouse/position.rs` | Position struct, `liquidationPx`, margin calculations |
| `l1/src/clearinghouse/position_map.rs` | Position map, position lookup |
| `l1/src/clearinghouse/margin_table.rs` | `MarginTable`, `RawMarginTier`, margin tiers |
| `l1/src/clearinghouse/oracle/oracle.rs` | Oracle prices for mark/liquidation |
| `l1/src/clearinghouse/impl_first_dex.rs` | First DEX clearinghouse implementation |
| `l1/src/bole/user_state.rs` | BOLE lending user state |
| `l1/src/exchange/exchange.rs` | Exchange struct (57 fields) |
| `l1/src/exchange/end_block.rs` | `begin_block` pipeline |
| `l1/src/dex_registry/clearinghouses.rs` | Multi-DEX clearinghouse registry |
| `l1/src/spot_clearinghouse/cl_trait.rs` | Spot clearinghouse trait |
| `l1/src/spot_clearinghouse/spot_clearinghouse.rs` | Spot clearinghouse |

---

## 1. Liquidation Trigger Conditions

### Margin Checks
The clearinghouse tracks per-user margin via these API fields:

```
marginSummary {
    crossMarginSummary         // aggregate cross margin
    crossMaintenanceMarginUsed // maintenance margin consumed
    withdrawable               // free equity
    assetPositions             // per-asset positions
}
```

Key fields on each position:

```
liquidationPx     // price at which position gets liquidated
marginUsed        // margin consumed by this position
positionValue     // notional value
unrealizedPnl     // unrealized PnL
returnOnEquity    // ROE
maxLeverage       // max allowed leverage for this tier
cumFunding        // cumulative funding
```

### Liquidation Eligibility
- Field `check_all_users_for_liquidatables` on the clearinghouse tracks the full set of users that need liquidation checking
- `users_with_positions` tracks all users with open positions
- `margin_available` is computed; when negative, user is liquidatable
- Log: `"Position is not liquidatable: "` (rejection message)

### Margin Ratio
The `risk_key` field appears in the context of liquidation ordering (see `Deadlockoverflowrisk_key`). Positions are sorted by risk for processing priority.

---

## 2. Auto-Deleveraging (ADL)

### Key Fields (from `SealedBlock` / `SubAccountModifyAction` struct):
```
oracle                          // oracle prices
total_net_deposit               // total net deposits
total_non_bridge_deposit        // total non-bridge deposits
perform_auto_deleveraging       // bool: whether to perform ADL
adl_shortfall_remaining         // f64: remaining shortfall to ADL
bridge2_withdraw_fee            // bridge withdrawal fee
```

### ADL Data Structures
```rust
struct AdlRequirement {
    position_index,    // which position
    // margin table reference
}

// Per-asset ADL tracking
asset_and_side_to_adl_requirements  // Map<(asset, side), Vec<AdlRequirement>>
```

### ADL Ranking (CONFIRMED 2026-04-02)
**CONFIRMED**: ADL uses **ROE (Return on Equity) ranking** to select counter-party positions for deleveraging. Positions with highest ROE on the opposing side are deleveraged first. This was previously INFERRED but is now CONFIRMED from binary RE.

### ADL Log Messages
- `" immediate adl @@ [liquidatable: "` -- immediate ADL triggered for liquidatable position
- `"Auto-deleveragings triggered @@ [total_shortfall: "` -- ADL batch triggered
- `"Not performing auto-deleveraging because shortfall is acceptable @@ [total_shortfall: "` -- shortfall below threshold

### ADL Shortfall Fields
- `adl_shortfall_remaining` -- tracked on `SealedBlock`, remaining shortfall after liquidation
- `adl_shortfall_re` (truncated in binary) -- appears in Exchange struct context
- `total_shortfall` -- aggregate shortfall across all positions
- `"Adl shortfall too large"` -- error when ADL shortfall exceeds limits

### DailyScaledVlms ADL Field
The `DailyScaledVlms` struct has field `adlri` (ADL risk index) as one of its variant fields (variant index 0 <= i < 14).

---

## 3. Backstop Liquidation

### Backstop Liquidator Types
```rust
enum BackstopLiquidatorMode { ... }  // Hip3BackstopLiquidatorMode

struct BackstopLiquidatorParams { ... }
enum BackstopLiquidatorParams { ... }  // also used as enum

struct BackstopLiquidatorTokenParams { ... }  // per-token config

struct Hip3BackstopLiquidatorParams { ... }
enum Hip3BackstopLiquidatorParams { ... }
```

### SetBole Governance Variant
```rust
struct variant SetBole::BackstopLiquidatorTokenParams with 4 elements
```
This is how validators configure backstop liquidator parameters per token.

### Hip3 Backstop
```rust
Hip3BackstopLiquidatorMode     // enum controlling backstop mode for HIP-3 assets
Hip3BackstopLiquidatorParams   // parameters struct/enum
```

Governance action:
```
struct variant VoteGlobalAction::SetHip3BackstopLiquidatorParams
```

### LtHash Category
`BoleBackstopLiquidation` is one of the LtHash update categories (alongside `BoleMarketLiquidation`, `BolePartialLiquidation`).

---

## 4. Partial Liquidation

### Exchange State Field
```
partial_liquidation_cooldown    // on Exchange struct (field ~index 40+)
```
This cooldown prevents rapid re-liquidation of partially liquidated positions.

### API Response Field
```
partialLiquidatable   // bool in user state / clearinghouse response
```

### LtHash Category
`BolePartialLiquidation` -- state hash update for partial liquidation events.

### Hip3 Partial Liquidation
HIP-3 assets have their own partial liquidation handling with configurable limits:
```
max_slippage        // max allowed slippage during partial liquidation
half_life           // decay parameter for partial liquidation sizing
```

---

## 5. Market Liquidation

### Order Type
```rust
enum OrderTypeWire {
    // variant index 0 <= i < 5:
    //   Alo, Ioc, Gtc, FrontendMarket, LiquidationMarket
}
```

`LiquidationMarket` is a special order type used exclusively for liquidation fills. It bypasses normal order validation.

### Trade Event Fields
```
liquidation         // field on fill/trade events
liquidatedUser      // which user was liquidated
markPx              // mark price at liquidation
feeTrialEscrow      // fee trial escrow handling
liquidatedCanceled  // order status: canceled due to liquidation
```

### Liquidation Types (from fills)
```
closedPnl           // PnL realized from liquidation close
crossedfee          // fee on crossed liquidation
builderFee          // builder fee (if applicable)
deployerFee         // deployer fee (HIP-3)
```

### LtHash Category
`BoleMarketLiquidation` -- state hash update for market liquidation events.

### Related Order Statuses
Orders can be force-canceled during liquidation:
```
liquidatedCanceled           // order canceled because user was liquidated
vaultWithdrawalCanceled      // canceled due to vault withdrawal
openInterestCapCanceled      // canceled at OI cap
selfTradeCanceled            // self-trade prevention
reduceOnlyCanceled           // reduce-only condition
siblingFilledCanceled        // sibling order filled
outcomeSettledCanceled       // outcome settled (HIP-4)
scheduledCancel              // scheduled cancel
internalCancel               // internal cancel
```

---

## 6. Margin Tiers

### Data Structures
```rust
struct RawMarginTier {
    lowerBound,    // position size lower bound for this tier
    maxLeverage,   // max leverage allowed at this tier
}

struct RawMarginTable {
    marginTiers,   // Vec<RawMarginTier>
}

struct MarginTable { ... }         // internal representation
struct MarginTableSerde { ... }    // serialization format (tuple)
struct MarginTierSerde {
    usingBigBlocks,   // bool flag for big block support
}

tuple struct MarginTableId         // newtype for table ID
```

### Internal MarginTable Fields
```
lower_bound              // f64: lower bound of tier
max_leverage             // f64: max leverage for tier
maintenance_deduction    // f64: maintenance margin deduction
margin_tiers             // Vec of tier definitions
```

### Governance Actions
```
registerAsset            // register new asset (includes margin table)
insertMarginTable        // insert a new margin table
setMarginTableIds        // assign margin tables to assets
```

### API Response
```
marginTables {
    collateralToken,
    szDecimals,
    maxLeverage,         // per-asset max leverage
    marginTableId,       // which margin table applies
    onlyIsolated,        // bool: isolated-only asset
    isDelisted,          // bool: delisted
    marginMode,          // Cross or Isolated
    growthMode,          // growth mode
    lastGrowthModeChangeTime,
}
```

### Error Messages
- `"Margin table missing"` -- asset has no margin table configured
- `"Margin table description too long"` -- margin table desc length limit
- `"Exceeded max number of margin tables."` -- max tables limit reached
- `"Invalid margin table id"` -- invalid table ID reference
- `". Manually set leverage to acknowledge new margin tiers. Alternatively, reduce position using reduce-only orders or add collateral to meet initial margin requirement."` -- user must acknowledge new tiers
- `"Cannot automatically accept new margin tiers on "` -- auto-accept blocked

---

## 7. Cross vs Isolated Margin

### MarginMode Enum
```rust
enum MarginMode {
    // variant index 0 <= i < 3:
    //   enabled, blocked, noCross, strictIsolated
    // Actually appears as: enabled | blocked | noCross | strictIsolated
}
```

Wait -- from the binary, the actual variants are:
```
enabled
blocked
noCross
strictIsolated
```

### Actions
```
UpdateIsolatedMargin           // Action: update isolated margin
TopUpStrictIsolatedMargin      // Action: top up strict isolated margin
UpdateUserLeverage             // Action: change leverage
```

Wire-level names:
```
updateIsolatedMargin           // camelCase in wire format
topUpIsolatedOnlyMargin        // alternative name in API
updateLeverage                 // leverage change
```

### Key Fields
- `isCross` -- boolean on position/order indicating cross margin
- `strictIsolated` -- strict isolated mode (cannot switch to cross)
- `onlyIsolated` -- asset-level flag: only isolated margin allowed
- `isolated_external` -- external isolation mode
- `isolated_oracle` -- oracle mode for isolated positions
- `hip3_isolated_on` -- HIP-3 isolated mode flag
- `hip3_no_cross` -- HIP-3 no-cross-margin flag

### Error Messages
- `"Cross margin is not allowed for this asset."` -- asset restricted to isolated
- `"Cannot switch leverage type with open position."` -- cannot change cross<->isolated with positions
- `"Cross position does not have sufficient margin available to decrease leverage. To decrease leverage, deposit more collateral."` -- cross margin too low
- `"Isolated position does not have sufficient margin available to decrease leverage. To decrease leverage, add margin to the position."` -- isolated margin too low
- `"Unknown user while updating isolated margin."` -- unknown user
- `"Update to isolated margin cannot be zero."` -- zero update rejected
- `"Cannot update margin for empty position."` -- no position
- `"Account does not have sufficient margin available for increase."` -- margin increase rejected
- `"Position does not have sufficient margin for reduction."` -- margin reduction rejected
- `"Position does not use isolated margin."` -- cross position
- `"Cannot remove isolated margin for this asset. To remove margin, reduce the position."` -- removal blocked
- `"Liquidator must use cross margin for asset "` -- liquidator cross-margin requirement
- `"Asset not strict isolated"` -- asset is not strict isolated
- `"internal error: entered unreachable code: Position must be cross"` -- assertion: position must be cross

---

## 8. Liquidation Ordering

### Position Index
```
position_index    // used for ordering within ADL
```

The `AdlRequirement` struct contains `position_index`, linking ADL requirements to specific positions.

### Risk Key
`risk_key` appears in the context of `Deadlockoverflowrisk_key`, suggesting positions are sorted by a risk metric for liquidation priority.

### Liquidation Pipeline (from `begin_block`, CONFIRMED 9 effects 2026-04-02)
The `begin_block` function runs this pipeline (VA 0x01e748e0):
```
1. update_oracle                  // update oracle prices from validator votes
2. distribute_funding             // distribute pending funding
3. apply_bole_liquidations        // BOLE lending liquidations
4. update_funding_rates           // update funding rates
5. refresh_hip3_stale_mark_pxs    // refresh HIP-3 mark prices (10s stale window)
6. prune_book_empty_user_states   // cleanup empty states
7. update_staking_rewards         // stage and distribute staking rewards
8. update_action_delayer (?)      // older RE placement; exact binary slot still open
9. update_aligned_quote_token     // sample aligned quote token prices
```

`apply_bole_liquidations` runs as part of every block's begin_block, before trading.
Current local `main` drains matured delayed actions after the 9 standard
begin-block effects; the exact binary placement of the delayed-action lane
relative to BOLE and the rest of the begin-block body is still open.

### Position Side Check
```
// position opening / direction strings:
"Open Long"
"Open Short"
"Short > Long"
"Long > Short"
"Close Short"
"Close Long"
```

---

## 9. Default Liquidator

### Fields
```
defaultLiquidator     // on multiple structs: Trigger, UsdcNtl, and borrow/lend actions
```

Found in these contexts:
```rust
// On Trigger struct:
struct Trigger {
    defaultLiquidator,
    request,            // LiquidateRequest
    // field index 0 <= i < 3
}

// On UsdcNtl:
tuple struct UsdcNtl {
    defaultLiquidator,
    l,                  // ?
    raw_usd,
    u,
}

// On borrow/lend actions:
// variants: optOut, borrow, supply, repay, defaultLiquidator
// These are the operations in the BOLE lending system
```

### Allowed Liquidators
```
allowed_liquidators    // field on Exchange struct (Clearinghouse level)
owed_liquidators       // liquidators that are owed fees/positions
```

The Exchange struct contains `allowed_liquidators` as a top-level field in the clearinghouse section.

### Governance
```
struct variant VoteGlobalAction::UserCanLiquidate  // governance: allow user to liquidate
```

### Error Messages
- `"Liquidator does not have enough margin available."` -- liquidator insufficient margin
- `"Invalid liquidator"` -- invalid liquidator address
- `"Default liquidator included as additional liquidator"` -- cannot duplicate default
- `"  is assigned to liquidator twice"` -- duplicate liquidator assignment

---

## 10. LiquidateRequest Enum

### Variants
```rust
enum LiquidateRequest {
    Cross {
        liquidator_to_assets,  // Map<User, Vec<Asset>>
    },
    Isolated {
        liquidator_to_assets,  // Map<User, Vec<Asset>>
        asset,                 // single asset for isolated
    },
}
```

From Ghidra struct variants:
```
struct variant LiquidateRequest::Cross with 1 element
    - liquidator_to_assets

struct variant LiquidateRequest::Isolated with 1 element
    - liquidator_to_assets, asset
```

### LiquidateAction
```rust
struct LiquidateAction {
    // from Action enum: variant "Liquidate" / "liquidate"
    // contains a LiquidateRequest
}
```

Wire format (within Action enum, variant index among 80 total actions):
```
// CamelCase (wire):  Liquidate
// camelCase (API):   liquidate
```

### Fill Event for Liquidation
When a liquidation fills, the trade event includes:
```
px              // execution price
sz              // execution size
startPosition   // position before
dir             // direction
closedPnl       // realized PnL
oid             // order ID
crossed         // was order crossed
fee             // fee charged
builderFee      // builder fee
tid             // trade ID
cloid           // client order ID
liquidation     // liquidation flag (bool)
feeTrialEscrow  // fee trial
builder         // builder address
twapId          // TWAP ID (if applicable)
deployerFee     // deployer fee (HIP-3)
liquidatedUser  // user who was liquidated
markPx          // mark price at liquidation
```

---

## LtHash Liquidation Categories

These are the state hash update operations for liquidation events (from `ConciseLtHash` / LtHash pipeline):

```
Na                           // No operation
LiquidatedCross              // Cross margin liquidation
LiquidatedIsolated           // Isolated margin liquidation
Settlement                   // Settlement
NetChildVaultPositions       // Child vault position netting
SpotDustConversion           // Spot dust conversion
BoleMarketLiquidation        // BOLE lending market liquidation
BoleBackstopLiquidation      // BOLE lending backstop liquidation
BolePartialLiquidation       // BOLE lending partial liquidation
```

---

## BOLE Lending Liquidation (Borrow/Lend)

### Order Types for BOLE Liquidations
```
borrowLendLiquidation    // special order type for borrow/lend liquidations (variant of OrderType)
"Borrow Liquidation"     // display name
```

### BoleUserState
Per-user BOLE lending state:
```
BoleUserState  // struct tracked on user
```

### Health Factor
```
health           // health factor
healthFactor     // API name
borrow           // borrow amount
supply           // supply amount
balance          // balance
utilization      // utilization ratio
ltv              // loan-to-value
totalSupplied    // total supplied
totalBorrowed    // total borrowed
basis            // basis
value            // value
```

### Reserve Accumulators
```
reserve_accumulators    // on Exchange struct -- tracks BOLE reserve accumulation
```

### Rate Model
```rust
struct RawRateCurve {
    // 4 fields:
    // ty, u, bus, MpmpP  (obfuscated field names)
    kink_rate,          // interest rate at utilization kink
    kink_utilization,   // utilization threshold
}
```

---

## Vault Liquidation Rules

- `"Cannot deposit into liquidatable vault."` -- deposits blocked for liquidatable vaults
- `"Cannot withdraw from liquidatable vault."` -- withdrawals blocked for liquidatable vaults
- Vault positions tracked via `liquidatedPositions` in fill events
- `"Vault withdrawal over daily limit."` -- daily withdrawal limit
- `"Invalid max HLP withdraw fraction"` -- HLP withdrawal fraction limit
- `max_hlp_withdraw_fraction_per_day` -- Exchange field
- `hlp_start_of_day_account_value` -- for daily limit calculation

---

## Exchange Struct Fields (Liquidation-Related)

From the Exchange struct (57 fields total):
```
locus                        // exchange locus
perp_dexs                    // perp DEX instances
spot_books                   // spot order books
agent_tracker                // agent tracking
funding_distribute_guard     // funding distribution guard
funding_update_guard         // funding update guard
sub_account_tracker          // sub-account tracking
allowed_liquidators          // Map of allowed liquidators
bridge2                      // bridge v2
staking                      // staking state
c_staking                    // CStaking
funding_err_dur_guard        // funding error duration guard
max_order_distance_from_anchor // max order distance
scheduled_freeze_height      // scheduled freeze
simulate_crash_height        // simulate crash
validator_power_updates      // validator power updates
cancel_aggressive_orders_at_open_interest_cap_guard // OI cap guard
last_hlp_cancel_time         // last HLP cancel time
post_only_until_time         // post-only mode
post_only_until_height       // post-only height
spot_twap_tracker            // spot TWAP tracker
user_to_display_name         // display names
book_empty_user_states_pruning_guard // pruning guard
user_to_scheduled_cancel     // scheduled cancels
hyperliquidity               // HLP state
spot_disabled                // spot disabled flag
prune_agent_idx              // agent pruning
max_hlp_withdraw_fraction_per_day // HLP daily limit
hlp_start_of_day_account_value // HLP daily value
register_token_gas_auction   // token registration auction
perp_deploy_gas_auction      // perp deploy auction
spot_pair_deploy_gas_auction // spot pair deploy auction
hyper_evm                    // HyperEVM state
vtg                          // vote tracker group
app_hash_vote_tracker        // app hash voting
begin_block_logic_guard      // begin_block guard
user_to_evm_state            // EVM state per user
multi_sig_tracker            // multi-sig tracking
reserve_accumulators         // BOLE reserve accumulators
partial_liquidation_cooldown // PARTIAL LIQUIDATION COOLDOWN
evm_enabled                  // EVM enabled flag
validator_l1_vote_tracker    // L1 vote tracking
staking_link_tracker         // staking link tracking
action_delayer               // action delayer
default_hip3_limits          // default HIP-3 limits
hip3_stale_mark_px_guard     // HIP-3 stale mark price guard
disabled_precompiles         // disabled EVM precompiles
lvt                          // ?
hip3_no_cross                // HIP-3 no cross-margin flag
initial_usdc_evm_system_balance // initial EVM USDC
validator_l1_stream_tracker  // L1 stream tracking
last_aligned_quote_token_sample_time // aligned quote token sampling
user_to_evm_to_l1_wei_remaining // EVM to L1 wei remaining
dex_abstraction_enabled      // DEX abstraction flag
hilo                         // ?
h                            // ?
abstraction                  // abstraction state
```

---

## Complete Liquidation Pipeline

```
Block N arrives
  |
  v
begin_block() -- CONFIRMED 9 effects (VA 0x01e748e0)
  |-- update_oracle()               // 1. update oracle prices
  |-- distribute_funding()          // 2. distribute pending funding
  |-- apply_bole_liquidations()     // 3. check & execute BOLE liquidations
  |-- update_funding_rates()        // 4. update funding rates
  |-- refresh_hip3_stale_mark_pxs() // 5. refresh HIP-3 mark prices (10s window)
  |-- prune_book_empty_user_states() // 6. cleanup
  |-- update_staking_rewards()      // 7. stage/distribute staking rewards
  |-- update_action_delayer() (?)   // older RE placement; exact binary slot still open
  |-- update_aligned_quote_token()  // 9. aligned quote token sampling
  |
  v
User actions processed (if any):
  |-- Action::Liquidate(LiquidateAction)
  |   |-- LiquidateRequest::Cross { liquidator_to_assets }
  |   |-- LiquidateRequest::Isolated { liquidator_to_assets, asset }
  |   |
  |   |-- Check: margin_available < 0 (user is liquidatable)
  |   |-- Check: Liquidator has enough margin
  |   |-- Check: Liquidator uses cross margin for asset
  |   |-- Check: Liquidator is in allowed_liquidators (or is default)
  |   |
  |   |-- Execute: LiquidationMarket order placed
  |   |-- Events: "Liquidated Cross " / "Liquidated Isolated "
  |   |
  |   |-- If shortfall remains: ADL triggered
  |       |-- perform_auto_deleveraging = true
  |       |-- adl_shortfall_remaining updated
  |       |-- Positions sorted by position_index + risk_key
  |       |-- Counter-party positions reduced
  |
  v
LtHash updated with category:
  |-- LiquidatedCross
  |-- LiquidatedIsolated
  |-- BoleMarketLiquidation
  |-- BoleBackstopLiquidation
  |-- BolePartialLiquidation
```

## Links
- [[BOLE]] -- BOLE lending protocol
- [[Clearinghouse]] -- Clearinghouse architecture
- [[Exchange State]] -- Exchange struct
- [[Action Types]] -- Action enum variants
- [[LtHash]] -- LtHash state hash categories
- [[Matching Engine]] -- Order types including LiquidationMarket
