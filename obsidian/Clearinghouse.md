# Clearinghouse

Manages user positions, margin, and liquidations in [[Hyperliquid]]. **18 fields CONFIRMED** (2026-04-02).

## Fields (18 fields, CONFIRMED from binary RE + ABCI snapshot)

The Clearinghouse struct has two sub-components at the top: `meta` (PerpMeta, 6 fields) and `user_states` (UserStates, 6 fields), followed by 12 named fields, plus 3 short-code fields at the end. Total: 18 elements as stated by the binary descriptor.

### Sub-components
| Sub-component | Type | Fields |
|---------------|------|--------|
| `meta` | PerpMeta(6) | marginTableIdToMarginTable, collateralToken, collateralIsAlignedQuoteToken, collateralTokenName, + 2 more |
| `user_states` | UserStates(6) | user state tracking (6 elements per binary descriptor) |

### Named Fields (15 visible, 3 short-code)
| # | Field | Type | Purpose |
|---|-------|------|---------|
| 1 | `meta` | PerpMeta(6) | Margin tables, collateral token config |
| 2 | `user_states` | UserStates(6) | Per-user position/margin state |
| 3 | `oracle` | Oracle(4) | pxs, external_perp_pxs, err_dur_guard, lpxk |
| 4 | `total_net_deposit` | f64 | Total USDC deposited |
| 5 | `total_non_bridge_deposit` | f64 | Non-bridge deposits |
| 6 | `perform_auto_deleveraging` | bool | ADL enabled flag |
| 7 | `adl_shortfall_remaining` | f64 | ADL shortfall tracking |
| 8 | `bridge2_withdraw_fee` | f64 | Bridge withdrawal fee |
| 9 | `daily_exchange_scaled_and_raw_vlms` | DailyScaledVlms | Volume tracking |
| 10 | `halted_assets` | HashSet\<u32\> | Per-asset trading halt |
| 11 | `override_max_signed_distances_from_oracle` | HashMap\<u32, f64\> | Oracle distance override |
| 12 | `max_withdraw_leverage` | f64 | Max leverage for withdrawal |
| 13 | `last_set_global_time` | DateTime\<Utc\> | Last governance action time |
| 14 | `usdc_ntl_scale` | (f64, u64) | USDC notional scaling -- tuple of float + integer |
| 15 | `isolated_external` | ? | External isolated margin |
| 16 | `isolated_oracle` | ? | Isolated oracle prices |
| 17 | `moh` | MainOrHip3 enum | **CONFIRMED**: discriminant for main clearinghouse vs HIP-3 clearinghouse |
| 18 | `znfn` | u64 | **CONFIRMED**: fill nonce counter (zero-based next fill nonce) |

## User State
From state JSON: `user_states.user_to_state`
```json
{
  "u": 10269381274,         // USDC balance (raw, /1e6 = dollars)
  "p": {
    "p": [                   // positions array
      [0, {                  // asset 0 = BTC
        "l": {"C": 15},     // cross leverage 15x
        "M": 56,            // margin table ID 56
        "f": {"a": -217821} // accumulated funding
      }]
    ]
  }
}
```

## Margin System
Tiered margin — different leverage at different position sizes:
- **MarginTier(3)**: lower_bound, max_leverage, maintenance_deduction
- **MarginTable(3)**: margin_tiers array
- Per-asset margin tables set by `insertMarginTable` governance
- Margin modes: normal, noCross, strictIsolated

### Liquidation
- `check_all_users_for_liquidatables` — scan field
- `partial_liquidation_cooldown` — spacing between liquidations
- `allowed_liquidators` — whitelist
- `perform_auto_deleveraging` + `adl_shortfall_remaining` — ADL system
- Liquidation condition: `account_value < maintenance_margin`

## Funding ([[Funding]])
- `FundingTracker(8)`: asset_to_premiums, override_impact_usd, clamp, default_impact_usd, hl_only_perps, use_binance_formula, perp_to_funding_multiplier, perp_to_funding_interest_rate
- Per-position funding accumulator: `f.a` in position data
- Distributed via `funding_distribute_guard` bucket guard
- **CONFIRMED**: Funding settlement period = 3600 seconds (1 hour)

## MainOrHip3 Enum (moh field, CONFIRMED 2026-04-02)
The `moh` field discriminates between the main clearinghouse and HIP-3 deployed clearinghouses:
- `cls[0]` = Main clearinghouse (moh = Main variant)
- `cls[1-8]` = HIP-3 clearinghouses (moh = Hip3 variant)
This determines which fee schedules, limits, and oracle update rules apply.

## Our Implementation
`hl-engine/src/clearinghouse.rs`:
- `resolve_price()` — mark > oracle > entry
- `account_value()` — balance + unrealized PnL
- `total_notional()` — sum of all position notionals
- `total_maintenance_margin()` — tiered computation
- `is_liquidatable()` — account_value < maintenance
- `process_fill()` — weighted-average entry price updates

## Links
- [[Exchange State]] — clearinghouse lives in `locus.cls[]`
- [[Matching Engine]] — fills update positions
- [[Funding]] — funding rates applied to positions
- [[Guards and Limits]] — liquidation cooldown

#positions #margin #liquidation
