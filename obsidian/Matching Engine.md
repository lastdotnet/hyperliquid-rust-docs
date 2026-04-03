# Matching Engine

The core order matching system in [[Hyperliquid]]. Handles order placement, cancellation, modification, fill generation, TWAP slicing, trigger orders, and builder fees.

## Original Source Structure (from binary RE)

```
l1/src/book/book/mod.rs            — Book struct, book management
l1/src/book/book/impl_insert_order.rs — core order insertion + matching
l1/src/book/book_orders.rs          — BookOrders tuple struct wrapper
l1/src/books.rs                     — multi-book manager (Books)
l1/src/exchange/impl_trading.rs     — top-level trading action dispatch
l1/src/exchange/exchange.rs         — Exchange struct (owns all state)
l1/src/exchange/end_block.rs        — end-of-block processing
l1/src/order_wire.rs                — OrderWireSerde, wire format parsing
l1/src/dex_registry/perp_dex.rs     — PerpDex: books + funding + twap + annotations
l1/src/clearinghouse/clearinghouse.rs — margin, positions, liquidation
l1/src/clearinghouse/user_state.rs  — per-user margin/position state
l1/src/clearinghouse/user_states.rs — user state collection
l1/src/clearinghouse/position.rs    — position math
l1/src/fees/compute.rs              — fee computation
l1/src/hyperliquidity.rs            — HLP market making
```

## Data Structures (from serde debug strings)

### Exchange (top-level state)
```rust
struct Exchange {
    // Book layer
    locus: Locus,           // per-asset book state container
    // ... 60+ fields including:
    post_only_until_time: Option<u64>,
    post_only_until_height: Option<u64>,
    cancel_aggressive_orders_at_open_interest_cap_guard: BucketGuard,
    last_hlp_cancel_time: u64,
    max_order_distance_from_anchor: f64,
    user_to_scheduled_cancel: HashMap<Address, ScheduledCancel>,
    max_hlp_withdraw_fraction_per_day: f64,
    partial_liquidation_cooldown: ...,
    spot_disabled: bool,
    hip3_no_cross: bool,
}
```

### Locus (15 fields -- per-asset book container)
```rust
struct Locus {
    perp_dexs: Vec<PerpDex>,       // one per perp asset
    spot_books: Vec<SpotBook>,     // one per spot pair
    agent_tracker: AgentTracker,
    funding_distribute_guard: BucketGuard,
    funding_update_guard: BucketGuard,
    sub_account_tracker: SubAccountTracker,
    allowed_liquidators: Vec<Address>,
    bridge2: Bridge2,
    staking: Staking,
    c_staking: CStaking,
    funding_err_dur_guard: ErrDurGuard,
    max_order_distance_from_anchor: f64,
    scheduled_freeze_height: Option<u64>,
    simulate_crash_height: Option<u64>,
}
```

### PerpDex (4 fields)
```rust
struct PerpDex {
    books: Books,                    // the actual order book
    funding_tracker: FundingTracker, // 8 fields
    twap_tracker: TwapTracker,       // 2 fields
    perp_to_annotation: HashMap<u32, PerpAnnotation>, // 5 fields
}
```

### BookDe (8 fields -- serialized Book)
```rust
struct BookDe {
    halfs: [Half; 2],               // Bid half + Ask half
    oid_to_key: HashMap<u64, BookKey>,
    cloid_tracker: CloidTracker,     // 1 field
    book_orders: BookOrders,         // tuple struct wrapping Vec
    asset: u32,
    user_states: UserStates,         // 6 fields
    last_trade_px: f64,
    open_order_tracker: OpenOrderTracker, // 1 field
}
```

### Half (1 field -- one side of the book)
```rust
struct Half {
    levels: BTreeMap<PriceKey, PriceLevel>,  // sorted price levels
}
// "B" = Bid, "A" = Ask  (from RawVlm: BBidAAsk)
```

### BookOrder (9 fields)
```rust
struct BookOrder {
    oid: u64,                // unique order ID
    order: OrderCore,        // core order fields
    resting_sz: f64,         // "O" prefix in serialization
    orig_sz: f64,            // original submitted size
    timestamp: u64,          // "R" prefix -- time placement
    relationships: Relationships, // parent/sibling/above/below refs
    // ... other tracking fields
}
```

### OrderCore
```rust
struct OrderCore {
    // Inferred from wire format fields:
    asset: u32,              // "a" field
    is_buy: bool,            // "b" field
    limit_px: f64,           // "p" field (limitPx on wire)
    sz: f64,                 // "s" field
    reduce_only: bool,       // "r" field
    order_type: OrderTypeWire, // "t" field
    cloid: Option<Cloid>,    // "c" field
}
```

## Order Types

### Tif (Time-in-Force) -- 3 variants
```rust
enum Tif {
    Alo,              // Add Liquidity Only (post-only)
    Ioc,              // Immediate or Cancel
    Gtc,              // Good Till Cancelled
}
```

### OrderTypeWire -- 2 variants
```rust
enum OrderTypeWire {
    Limit { tif: Tif },                           // "limit"
    Trigger { trigger_px: f64, is_market: bool, tpsl: Tpsl },  // "trigger"
}
// variant index 0 <= i < 2
```

### Tpsl enum -- 3 variants
```rust
enum Tpsl {
    Na,                // not a TP/SL
    NormalTpsl,        // "normalTpsl" -- normal TP/SL attached to order
    PositionTpsl,      // "positionTpsl" -- position-level TP/SL
}
// variant index 0 <= i < 3
```

### OrderDescription -- 10 variants (internal order origin)
```rust
enum OrderDescription {
    Limit,                   // "l" -- user limit order
    Market,                  // "m" -- market order (internally IOC limit)
    TakeProfitLimit,         // "T" -- TP limit trigger (Take Profit Limit)
    StopMarket,              // "S" -- stop-loss market
    StopLimit,               // "" -- stop-loss limit
    CloseForVaultWithdrawal, // "d" -- vault auto-close
    SpotDustConversion,      // "v" -- dust sweep (Spot Dust Conversion)
    BorrowLendLiquidation,   // "b" -- borrow/lend liquidation
    Liquidation,             // "" -- (LiquidationMarket / FrontendMarket)
    TwapSlice,               // "w" -- TWAP child order
}
// variant index 0 <= i < 10
// Single-char codes: l, m, T, S, (none), d, v, b, (none), w
```

### Grouping -- 4 variants (order grouping mode)
```rust
enum Grouping {
    Na,                // "na" -- default, no special grouping
    // Found in binary near twapId:
    Default,           // "default"
    DexAbstraction,    // "dexAbstraction"
    UnifiedAccount,    // "unifiedAccount"
    PortfolioMargin,   // "portfolioMargin"
}
// Note: "na" appears as the primary grouping value; others may be
// abstraction-layer grouping for DEX abstraction / portfolio margin features
```

### Side -- 2 variants
```rust
enum Side {
    Bid,  // "B" / "A" -- Buy
    Ask,  // "A" / "B" -- Sell
}
// Serialized as "BBid" "AAsk" in the binary
```

## Wire Format Structs

### OrderWireSerde (7 fields)
```rust
struct OrderWireSerde {
    a: u32,                  // asset index
    b: bool,                 // isBuy
    p: String,               // limitPx (decimal string)
    s: String,               // sz (decimal string)
    r: bool,                 // reduceOnly
    t: OrderTypeWire,        // orderType (limit or trigger)
    c: Option<Cloid>,        // cloid (client order ID)
}
```

### OrderAction (3 fields)
```rust
struct OrderAction {
    orders: Vec<OrderWireSerde>,       // the orders to place
    builder: Option<OrderBuilderInfo>, // builder fee info
    grouping: Grouping,               // "na" default
}
```

### OrderBuilderInfo (2 fields)
```rust
struct OrderBuilderInfo {
    b: String,   // builder address
    f: u64,      // builder fee (bps? or raw?)
}
```

### TriggerWireSerde (3 fields)
```rust
struct TriggerWireSerde {
    triggerPx: String,   // trigger price
    isMarket: bool,      // execute as market when triggered
    tpsl: Tpsl,          // TP/SL type
}
```

### TriggerOrder (2 fields)
```rust
struct TriggerOrder {
    above: Option<Trigger>,   // trigger when price goes above
    below: Option<Trigger>,   // trigger when price goes below
}
```

### Trigger (3 fields)
```rust
struct Trigger {
    triggerPx: f64,
    isMarket: bool,
    tpsl: Tpsl,
}
```

### ModifyWire (2 fields)
```rust
struct ModifyWire {
    oid: u64,           // order to modify
    order: OrderWireSerde,  // new order params
}
```

### Cancel (2 fields)
```rust
struct Cancel {
    a: u32,    // asset
    o: u64,    // oid
}
```

### CancelByCloid (2 fields)
```rust
struct CancelByCloid {
    asset: u32,
    cloid: Cloid,
}
```

## TWAP System

### TwapWireSerde (6 fields)
```rust
struct TwapWireSerde {
    a: u32,              // asset
    b: bool,             // isBuy
    s: String,           // sz (total)
    r: bool,             // reduceOnly
    m: u64,              // minutes (duration)
    t: bool,             // randomize (jitter)
}
```

### TwapTracker (2 fields)
```rust
struct TwapTracker {
    running_heap: BinaryHeap<TwapHeapKey>,  // priority queue by next slice time
    user_to_id_to_state: HashMap<Address, HashMap<TwapId, TwapSchedule>>,
}
```

### TwapSchedule (7 fields)
```rust
struct TwapSchedule {
    next_slice_time: u64,
    twap_id: TwapId,
    executed_sz: f64,
    executed_ntl: f64,
    slice_number: u64,
    // + 2 more fields (asset, direction info)
}
```

### TwapHeapKey (3 fields)
```rust
struct TwapHeapKey {
    next_slice_time: u64,  // when next slice fires
    twap_id: TwapId,       // unique TWAP ID
    // + 1 more field
}
```

### TWAP Validation Messages
```
"Exceeded max number of active TWAP orders."
"TWAP order value too small. Min is $..."
"TWAP order value too large. Max is $..."
"Reduce-only TWAP order would increase position."
"TWAP was never placed, already canceled, or filled."
```

## Order Status Lifecycle -- 35 variants

### Active States
| Status | Meaning |
|--------|---------|
| `resting` | On the book, waiting for match |
| `waitingForFill` | Submitted, pending execution |
| `waitingForTrigger` | Trigger order waiting for price |

### Terminal Positive
| Status | Meaning |
|--------|---------|
| `filled` | Fully filled |
| `triggered` | Trigger activated, converted to limit/market |

### Cancellation Reasons (engine-initiated)
| Status | Cause |
|--------|-------|
| `marginCanceled` | Insufficient margin after mark-to-market |
| `perpMaxPositionCanceled` | Would exceed max position size for leverage |
| `vaultWithdrawalCanceled` | Vault withdrawal forced close |
| `openInterestCapCanceled` | Asset OI cap reached |
| `selfTradeCanceled` | Self-trade prevention triggered |
| `reduceOnlyCanceled` | Reduce-only order would increase position |
| `siblingFilledCanceled` | OCO sibling was filled |
| `liquidatedCanceled` | User liquidated, orders cancelled |
| `outcomeSettledCanceled` | Prediction market outcome settled |
| `delistedCanceled` | Asset delisted |
| `scheduledCancel` | User-scheduled cancel (ScheduledCancel) |
| `internalCancel` | System-internal cancel |

### Rejection Reasons (pre-match validation)
| Status | Cause | Error Message |
|--------|-------|---------------|
| `tickRejected` | Price violates sig figs | "Price must be divisible by tick size." |
| `minTradeNtlRejected` | Below minimum notional | (impact_usd check) |
| `perpMarginRejected` | Insufficient margin | "Insufficient margin to place order." |
| `perpMaxPositionRejected` | Exceeds max position | "Order would exceed maximum position size for current leverage." |
| `reduceOnlyRejected` | Reduce-only would increase | "Reduce only order would increase position." |
| `iocCancelRejected` | IOC with no fills | "Order could not immediately match against any resting orders." |
| `badAloPxRejected` | Post-only would cross | "Post only order would have immediately matched." |
| `badTriggerPxRejected` | Invalid trigger price | "Invalid TP/SL price." |
| `marketOrderNoLiquidityRejected` | No resting orders | "No liquidity available for market order." |
| `positionIncreaseAtOpenInterestCapRejected` | OI cap, new position | "Cannot increase position when open interest is at cap." |
| `positionFlipAtOpenInterestCapRejected` | OI cap, flip | "Cannot flip position when open interest is at cap." |
| `tooAggressiveAtOpenInterestCapRejected` | OI cap, price too aggressive | "Asset open interest is at cap." |
| `openInterestIncreaseRejected` | OI increasing too fast | "Open interest is increasing too quickly. Try again in a few seconds." |
| `insufficientSpotBalanceRejected` | Not enough spot balance | "Insufficient spot balance" |
| `oracleRejected` | Price too far from oracle | "Price too far from oracle" |
| `outcomeSettledRejected` | Outcome already settled | "Outcome has been settled" |
| `hip3CrossMarginReduceOnly` | HIP-3 cross margin reduce-only | "Cross margin currently only supports reduce-only orders" |
| `hip3CrossMarginDisabled` | HIP-3 cross margin disabled | "Cross margin disabled" |

### Relationships (linked orders -- OCO/bracket)
```rust
struct Relationships {
    parent: Option<u64>,     // parent order OID
    sibling: Option<u64>,    // OCO sibling OID (one-cancels-other)
}
// When sibling is filled -> siblingFilledCanceled on the other
```

### OpenOrderTracker (1 field)
```rust
struct OpenOrderTracker {
    // Tracks per-user open order count
    // Enforces: "Too many open orders"
}
```

### ScheduledCancel struct
```rust
struct ScheduledCancel {
    // user_to_scheduled_cancel: HashMap<Address, ScheduledCancel>
    // "Too many scheduled cancels triggered for the day. Limit is ..."
    // "Cannot set scheduled cancel time until enough volume traded. Required: $..."
}
```

### Info API Response Fields (from clearinghouseState)
```
clearinghouseState, leadingVaults, openOrders, agentAddress, agentValidUntil,
cumLedger, assetCtxs, serverTime, isVault, twapStates, spotState,
spotAssetCtxs, optOutOfSpotDusting, perpsAtOpenInterestCap, userState,
perpDexStates, dexAbstractionEnabled, abstraction, description, displayName,
cumVlm, nRequestsUsed, nRequestsCap, marginSummary, crossMarginSummary,
crossMaintenanceMarginUsed, withdrawable, assetPositions, marginTables,
collateralToken, maxLeverage, marginTableId, onlyIsolated, isDelisted,
marginMode, growthMode, lastGrowthModeChangeTime
```

### Open Order Response Fields
```
limitPx, sz, oid, timestamp, isTriggered, triggerPx, isPosition, tpsl,
reduceOnly, orderType, origSz, tif, cloid
```

## Matching Pipeline (order lifecycle)

### 1. Wire Deserialization
```
OrderAction { orders: Vec<OrderWireSerde>, builder: Option<OrderBuilderInfo>, grouping }
  -> parse each OrderWireSerde into internal order representation
  -> validate: "Order has zero size", "Order has invalid price", "Order has invalid size"
  -> validate: "Order size cannot be larger than half of total supply"
  -> validate: "Invalid TIF", "Orders are empty", "Order action has different asset classes"
  -> validate: "A nonzero cloid is required for each order"
```

### 2. Pre-match Validation
```
Check 1: Trading halted?        -> "Trading is halted."
Check 2: Too many open orders?  -> "Too many open orders" (per-user limit)
Check 3: Tick size valid?       -> tickRejected ("Price must be divisible by tick size.")
Check 4: Oracle distance?       -> oracleRejected ("Price too far from oracle")
                                  + max_order_distance_from_anchor enforcement
                                  + "Price moved too far, new price is clamped"
Check 5: Margin sufficient?     -> perpMarginRejected
Check 6: Max position check?    -> perpMaxPositionRejected
Check 7: OI cap check?          -> positionIncreaseAtOpenInterestCapRejected
                                  + cancel_aggressive_orders_at_open_interest_cap_guard
Check 8: Reduce-only valid?     -> reduceOnlyRejected
Check 9: Spot balance?          -> insufficientSpotBalanceRejected
Check 10: Post-only period?     -> post_only_until_time / post_only_until_height
```

### 3. Matching (impl_insert_order.rs)
```
Price-time priority:
  - Bids sorted descending (negative keys in BTreeMap)
  - Asks sorted ascending (positive keys in BTreeMap)
  - At same price level: FIFO (VecDeque front = oldest)

For each crossing level:
  a. Self-trade check -> selfTradeCanceled (cancel resting order)
  b. Reduce-only check -> reduceOnlyCanceled if would increase
  c. Generate fill
  d. Update positions via clearinghouse
  e. Update OI tracking
```

### 4. Post-match
```
ALO: If any fills occurred -> badAloPxRejected (should not happen, checked pre-match)
IOC: Cancel remaining unfilled portion -> iocCancelRejected if zero fills
GTC: Rest remaining on the book -> status = resting
Trigger: Store in trigger tracker -> status = waitingForTrigger
```

### 5. Trigger Activation
```
When oracle/mark price crosses triggerPx:
  - Convert trigger to limit/market order
  - status: waitingForTrigger -> triggered -> re-enter matching pipeline
  - TriggerOrder has above/below triggers (TP = above for longs, SL = below)
  - Trigger orders MUST be reduce-only ("Trigger order is not reduce only.")
  - Trigger order side validated ("Trigger order has unexpected side.")
```

## Price System

### Tick Size / Sig Figs
- "Book sig figs must be 3, 4, or 5"
- Prices validated against tick size: "Price must be divisible by tick size."
- `OrderType::unflatten bad variant` -- error on invalid OrderType deserialization
- szDecimals: number of decimal places for size (per-asset)
- weiDecimals: number of decimal places for wei amounts
- "weiDecimals must be at least szDecimals + 5"
- "szDecimals too large"

### Price Clamping
- `max_order_distance_from_anchor` -- maximum distance from oracle/anchor price
- `clamped_price=` -- price is clamped when too far from oracle
- `default_impact_usd` / `override_impact_usd` -- impact price calculation
- `clamp` -- price clamping mechanism

## Open Interest Controls

### OI Caps (from Exchange fields)
```rust
// Per-asset OI caps
asset_to_oi_cap: HashMap<u32, f64>,        // total_oi_cap2
oi_sz_cap_per_perp: HashMap<u32, f64>,     // size-denominated
assets_at_open_interest_cap: HashSet<u32>,
asset_to_recent_oi: HashMap<u32, f64>,
override_max_oi_per_second: Option<f64>,
reset_recent_oi_bucket_guard: BucketGuard,
```

### OI Error Messages
```
"Cannot increase position when open interest is at cap."
"Cannot flip position when open interest is at cap."
"Open interest is increasing too quickly. Try again in a few seconds."
"Asset open interest is at cap."
"Asset size-denominated open interest is at cap."
"Perp DEX open interest is at cap."
```

## Self-Trade Prevention
- When incoming order would match against resting order from same user
- Resting order cancelled with `selfTradeCanceled` status
- Incoming order continues matching against other resting orders

## Reduce-Only Logic
- `reduceOnlyCanceled` -- resting reduce-only cancelled when position flips
- `reduceOnlyRejected` -- rejected at placement if would increase position
- "Reduce only order would increase position."
- "Reduce-only TWAP order would increase position."
- Trigger orders enforced reduce-only: "Trigger order is not reduce only."

## Builder Fee System
- `OrderBuilderInfo { b: String, f: u64 }` -- builder address + fee
- `ApproveBuilderFee` -- action to approve a builder's fee rate
- `SystemApproveBuilderFee` -- system-level builder approval
- `UserApproveBuilderFee` -- user approves builder fee
- `StartFeeTrial` -- start a fee trial period
- `approved_builder_fees` / `collected_builder_fees` -- tracked per-user

## Fee Structure

### FeeSchedule (9 fields)
```rust
struct FeeSchedule {
    // Cross/add rates per tier
    userCrossRate: f64,         // taker fee rate
    userAddRate: f64,           // maker fee rate (can be negative = rebate)
    userSpotCrossRate: f64,     // spot taker
    userSpotAddRate: f64,       // spot maker
    activeReferralDiscount: f64,
    trial: Option<FeeTrial>,
    feeTrialEscrow: f64,
    nextTrialAvailableTimestamp: u64,
    stakingLink: Option<StakingLink>,
    activeStakingDiscount: f64,
}
```

### Fee Tiers
```rust
struct FeeTiers {
    dp: ..., du: ...,      // cross/add rate structures
    // Perp fees
    perpCrossBaseRate: f64,
    perpAddBaseRate: f64,
    // Spot fees
    spotCrossBaseRate: f64,
    spotAddBaseRate: f64,
    // VIP/MM tiers
    vip: VipFees,          // { ntl_cutoff, rates, staking_keep_frac_for_referrer }
    mm: MmFees,            // { maker_fraction_cutoff, add }
}
```

## Scheduled Cancellation
```rust
struct ScheduledCancel {
    // Per-user scheduled cancel
    // Enforcement: "Too many scheduled cancels triggered for the day. Limit is ..."
    // Prerequisite: "Cannot set scheduled cancel time until enough volume traded. Required: $..."
}
```

## Modification
```rust
// BatchModifyAction { orders: Vec<ModifyWire> }
// ModifyWire { oid, order: OrderWireSerde }
// "Cannot modify canceled or filled order"
// "Attempted to modify to invalid new order"
```

## Liquidation Integration
- `LiquidationMarket` -- internal order type for liquidation fills
- `FrontendMarket` -- user-facing market orders
- Partial liquidation: `partial_liquidation_cooldown`
- ADL: "Auto-deleveragings triggered @@ [total_shortfall: ...]"
- `adl_shortfall_remaining` -- remaining shortfall after ADL
- Backstop liquidators: `allowed_liquidators`, `defaultLiquidator`
- "Liquidator does not have enough margin available."
- "Liquidator must use cross margin for asset"

## HyperLiquidity Provider (HLP)
```rust
struct Hyperliquidity {
    // Internal market maker
    // "insufficient usdc for seeding hyperliquidity"
    // "Hyperliquidity price is invalid"
    // "zero Hyperliquidity order size"
    // last_hlp_cancel_time, max_hlp_withdraw_fraction_per_day
    // hlp_start_of_day_account_value
}
```

## Action Types (all order-related)
From the `Action` enum (90 variants total, variant index 0 <= i < 90):
```
Order              -- place orders
Cancel             -- cancel by asset+oid
CancelByCloid      -- cancel by client order ID
BatchModify        -- modify multiple orders
TwapOrder          -- place TWAP
TwapCancel         -- cancel TWAP
ScheduleCancel     -- schedule future cancel
Modify             -- modify single order
Liquidate          -- liquidate a user
```

## Margin Modes
```rust
// From binary:
// "Cross margin is not allowed for this asset."
// "Cannot switch leverage type with open position."
// "Cross position does not have sufficient margin available to decrease leverage."
// "Isolated position does not have sufficient margin available to decrease leverage."

enum MarginMode {
    Enabled,         // "enabled"
    Blocked,         // "blocked"
    NoCross,         // "noCross"
    StrictIsolated,  // "strictIsolated"
}
```

## Spot-Specific
- `SpotDustConversion` -- automatic dust sweep
- `toggleSpotDusting` -- opt in/out of dust conversion
- `spot_disabled` flag on Exchange
- `spotDustConversion` order description variant
- Spot books separate from perp_dexs in Locus

## Performance
From live mainnet data:
- Peak: 2,451 actions in one 70ms block
- Peak second: 7,746 actions/second
- 200K+ operations/second (including internal matching)

## Our Implementation
`hl-engine/src/book.rs` + `hl-engine/src/matching.rs`:
- BTreeMap + VecDeque per price level (matches upstream Half + levels design)
- HashMap OID index for O(1) cancel (matches oid_to_key)
- 93 blocks/sec with 500K+ resting orders

## Links
- [[Exchange State]] -- perp_dexs, spot_books fields
- [[Clearinghouse]] -- position updates on fill
- [[Funding]] -- funding tracked per fill
- [[Guards and Limits]] -- max_order_distance_from_anchor
- [[Action Types]] -- order, cancel, modify actions
- [[LtHash Serialization]] -- book orders hashed into app_hash

#trading #orderbook #matching #reverse-engineering
