# Funding

Funding rate computation in [[Hyperliquid]] perp markets.

## Settlement Frequency
**CONFIRMED**: 3600 seconds (1 hour) -- confirmed from live API (fundingHistory timestamps are exactly 3600s apart) AND from binary RE.
NOT 8 hours like Binance. The API-reported `fundingRate` is the per-hour rate.

Default interest rate: 0.01% per 8h = 0.00125% per 1h = **0.0000125 per hour**.
From live API analysis: BTC pins to 0.0000125 in 314/500 hourly samples.

## Algorithm
Premium-based funding with fraction-based EMA smoothing:
1. `premium = (mark_px - oracle_px) / oracle_px`
2. EMA smoothing via [[DeterministicEma]](5) -- fraction-based: value = num/denom
3. Clamp smoothed premium to [-max_slippage, +max_slippage]
4. Apply per-asset multiplier: `perp_to_funding_multiplier`
5. Add interest rate component: `perp_to_funding_interest_rate`
6. Optional: Binance-style formula when `use_binance_formula=true`

### EMA Update (from ema_tracker.rs assertions)
```
decay = exp(-dt * ln(2) / half_life)   // assert!(decay <= 1.)
num = num * decay + sample * (1 - decay)
denom = denom * decay + (1 - decay)    // assert!(self.denom > 0.)
value = num / denom
```

### Binance Formula (when toggled)
```
premium = (impact_bid + impact_ask) / 2 - oracle_px
premium_index = premium / oracle_px
funding_rate = premium_index + clamp(interest - premium_index, -0.0005, 0.0005)
```

## Block Pipeline (begin_block) -- CONFIRMED 9 effects (2026-04-02)

**CONFIRMED from binary VA 0x01e748e0**: begin_block has **9 effects**, not 5. The full order is:

1. `update_oracle` -- update oracle prices from validator votes
2. `distribute_funding` -- settles accumulated funding to positions
3. `apply_bole_liquidations` -- BOLE liquidation pass
4. `update_funding_rates` -- samples premium, updates DeterministicEma per asset
5. `refresh_hip3_stale_mark_pxs` -- refreshes HIP-3 mark prices (10s stale window)
6. `prune_book_empty_user_states` -- cleans empty book/user state
7. `update_staking_rewards` -- stage and distribute staking rewards
8. `update_action_delayer` (?) -- older RE placement for delayed CoreWriter execution; exact binary slot still open
9. `update_aligned_quote_token` -- sample aligned quote token prices

The previous 5-effect model was a subset. The 4 additional effects (update_oracle,
update_staking_rewards, update_action_delayer, update_aligned_quote_token) were
confirmed as part of the widened begin-block / execution-state surface.

Important boundary: the repo's current local implementation drains matured
delayed actions after the 9 standard begin-block effects, while older RE notes
place `update_action_delayer` as effect 8 inside the begin-block body. Treat the
existence of the delayed-action lane as confirmed, but its exact placement
relative to BOLE and other begin-block effects as still open.

## FundingTracker (8 fields, serialization order confirmed)
1. `asset_to_premiums` -- **CORRECTED**: HashMap<u32, Vec<f64>> (NOT HashMap<u32, DeterministicEma>)
2. `override_impact_usd` -- HashMap<u32, f64>
3. `clamp` -- f64 (clamp factor)
4. `default_impact_usd` -- f64 (default: 10000)
5. `hl_only_perps` -- HashSet<u32>
6. `use_binance_formula` -- bool
7. `perp_to_funding_multiplier` -- HashMap<u32, f64>
8. `perp_to_funding_interest_rate` -- HashMap<u32, f64>

**CORRECTION 2026-04-02**: `asset_to_premiums` is `HashMap<u32, Vec<f64>>`, a vector of premium samples per asset, NOT a DeterministicEma per asset. The DeterministicEma is used in the volatility/mark price EMA pipeline, not for premium storage.

## DeterministicEma (5 fields -- FRACTION-BASED, CONFIRMED 2026-04-02)
1. `num` -- f64: weighted numerator
2. `denom` -- f64: weighted denominator (assert > 0)
3. `decay_duration` -- f64: time window
4. `n_samples` -- u64: sample count
5. `last_sample_time` -- **CONFIRMED**: String (ISO 8601 format, e.g. "2026-04-02T18:14:31.542650736Z")

**CONFIRMED**: The 5th field is `last_sample_time` as an ISO 8601 String, not a DateTime or u64 timestamp.
**half_life and max_slippage live on FeeTracker(10), not on DeterministicEma.**

## DeterministicVty (7 fields, CONFIRMED 2026-04-02)
1. `desc` -- String: description/name of this volatility tracker
2. `samples` -- Vec<(String, f64)>: timestamped samples (NOT VecDeque<f64>), each entry is (ISO8601 timestamp, value)
3. `emas` -- Vec<DeterministicEma>: vector of EMA trackers (one per configured half-life)
4. `val` -- Vty{min: f64, max: f64}: volatility bounds struct (NOT a plain f64)
5. `n_updates` -- u64: update counter
6. `n_bucket_updates` -- u64: bucket update counter
7. `alert_bucket_guard` -- BucketGuard: alert rate limiter

**CORRECTIONS 2026-04-02**:
- `samples` is `Vec<(String, f64)>` with ISO 8601 timestamps, not `VecDeque<f64>`
- `val` is a `Vty{min, max}` struct, not a plain f64
- `emas` is a Vec of DeterministicEma trackers, not a single EMA
- `desc` field added at the beginning
- `n_bucket_updates` is distinct from `n_updates`
- `bucket_limit_mult_threshold` was removed (lives on AlertConfig, not DeterministicVty)

## Oracle (4 fields)
1. `pxs` -- oracle prices per asset
2. `external_perp_pxs` -- external perp reference prices
3. `err_dur_guard` -- ErrDurGuard
4. `lpxk` -- OracleKindMap (last price kind)

## Mark Price Pipeline
Block inputs: `mark_px_inputs`, `spot_px_inputs`, `external_perp_px_inputs`, `oracle_pxs`
In-memory maps: `coin_to_mark_px`, `coin_to_oracle_px`, `coin_to_external_perp_px`

## Oracle Pipeline
External feeds -> ValidatorL1VoteAction -> VoteTracker (median) ->
SetGlobalAction -> Oracle -> mark_px computation

## Risk-Free Rate
StreamTracker with median of validator votes:
- `risk_free_rate` -- median_value from validator submissions
- `validator_to_value` -- per-validator rate votes
- `update_median_bucket_guard` -- rate limiter

## Configuration (via SetGlobalAction)
- `setFundingMultipliers` -- per-asset multiplier
- `setFundingInterestRates` -- per-asset interest rate
- `use_binance_formula` -- toggle Binance-style calculation
- `asset_to_premiums` -- premium tracking (Vec<f64> per asset, CORRECTED from DeterministicEma)
- `default_impact_usd` / `override_impact_usd` / `clamp` -- impact notional config
- `hl_only_perps` -- perps exclusive to HL (no external reference price)
- `perpCrossBaseRate` / `perpAddBaseRate` -- base funding rates
- `spotCrossBaseRate` / `spotAddBaseRate` -- spot base rates

## Rate Model
- `kink_rate` -- interest rate kink
- `kink_utilization` -- utilization at kink
- `min_rate` -- minimum rate

## Position-Level Funding
- `cumFunding` per position: since_open, since_change, duplicate
- funding_pnl = position_sz * (cum_funding_global - cum_funding_entry)
- Affects [[LtHash]] via "Settlement" operation

## Guards (BucketGuards)
- `funding_distribute_guard` -- rate limits distribution (hourly)
- `funding_update_guard` -- rate limits EMA updates
- `funding_err_dur_guard` -- error duration tracking

## Live API Observations
- Rate/premium ratio varies 0.03-0.07 (not constant: proves EMA dampening)
- Instantaneous premium of 0.001 -> settled rate ~0.00005
- Rate caps at 0.0000125 during low-premium periods (interest floor)
- No negative interest rate floor observed (rates can go negative)

## Source Paths (from binary)
- `/home/ubuntu/hl/code_Mainnet/base/src/ema_tracker.rs`
- `/home/ubuntu/hl/code_Mainnet/base/src/hour.rs`
- `/home/ubuntu/hl/code_Mainnet/base/src/cumin.rs` (cumulative minutes, assert <= 1440)

## Links
- [[Clearinghouse]] -- funding applied to positions
- [[Exchange State]] -- funding guards + trackers
- [[Guards and Limits]] -- bucket guards
- [[Matching Engine]] -- mark prices from order books

#funding #rates #defi
