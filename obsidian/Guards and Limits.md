# Guards and Limits

Protection mechanisms discovered in the [[Hyperliquid]] [[Binary Structure]].

## Rate Limiting

### Network Level
- `hard_rate_limit` / `soft_rate_limit`
- `rate_limiter_bytes_per_sec` — bandwidth cap
- `handle_request_rate_limiter` — per-request on info API
- `rate_limited_ips/` — tracked IPs
- `firewall_ips.rs` — IP-based firewall

### 16 BucketGuards
Internal rate limiters using time-bucketed windows:

| Guard | Limits |
|-------|--------|
| `funding_distribute_guard` | Funding distribution frequency |
| `funding_update_guard` | Funding rate update frequency |
| `begin_block_logic_guard` | Block processing reentrancy |
| `cancel_aggressive_orders_at_oi_cap_guard` | OI cap cancel rate |
| `book_empty_user_states_pruning_guard` | State pruning rate |
| `ensure_order_bucket_guard` | Order validation rate |
| `reset_recent_oi_bucket_guard` | OI tracking reset rate |
| `stakes_decay_bucket_guard` | Stake decay computation |
| `stage_reward_bucket_guard` | Reward staging |
| `distribute_reward_bucket_guard` | Reward distribution |
| `update_median_bucket_guard` | Median price update |
| `prune_bucket_guard` | General pruning |
| `alert_bucket_guard` | Alert rate |
| `referral_bucket_millis` | Referral system |
| `partial_liquidation_cooldown` | Liquidation spacing |
| `status_guard` | [[Action Delayer]] status check |

## Volume-Gated Features
Minimum trading volume required:
1. **Sub-accounts**: "Cannot create sub-accounts until enough volume traded"
2. **Scheduled cancels**: "Cannot set scheduled cancel time until enough volume traded"
3. **Portfolio margin**: "Cannot enable portfolio margin until enough volume traded"
4. **Extra agents**: "Too many extra agents for cumulative volume traded"
5. **Referral codes**: "Cannot generate referral code until enough volume traded"

## Order Validation
- Price bands: `max_order_distance_from_anchor`
- Size limits: half of total supply max
- Post-only: `badAloPxRejected` if would cross
- Sig figs: must be 3, 4, or 5
- Trading halt: per-asset `halted_assets`
- OI cap: `cancel_aggressive_orders_at_oi_cap`

## [[Action Delayer]]
9 fields controlling artificial delays:
```
delayed_actions, user_to_n_delayed_actions,
n_total_delayed_actions, max_n_delayed_actions,
vty_trackers, status_guard, enabled,
delayer_mode, delay_scale
```
Prevents latency advantages from [[HyperEVM]] → L1 operations.

## Cancel Flow (3 modes)
1. **Standard**: `{cancels: [{a, o}]}` — batch by asset + OID
2. **CancelByCloid**: client order ID cancel
3. **ScheduleCancel**: volume-gated, daily-limited timed cancel

## Links
- [[Exchange State]] — guards are fields in the 56-field struct
- [[Action Types]] — actions that hit the guards
- [[Matching Engine]] — order validation
- [[HyperEVM]] — disabled_core_writer_actions

#security #ratelimiting #protection
