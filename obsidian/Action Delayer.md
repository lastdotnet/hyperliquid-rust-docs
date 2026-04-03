# Action Delayer

Artificial delay system in [[Hyperliquid]] that prevents latency advantages, particularly for [[HyperEVM]] → L1 operations.

## Fields (9)
```
delayed_actions             — queue of pending actions
user_to_n_delayed_actions   — per-user delay count
n_total_delayed_actions     — global delay count
max_n_delayed_actions       — queue size limit
vty_trackers                — volatility trackers (DeterministicVty)
status_guard                — BucketGuard for status checks
enabled                     — master enable switch
delayer_mode                — which delay mode (Alternating, etc.)
delay_scale                 — configurable delay multiplier
```

## Modes
- `ActionDelayerStatus::Alternating(1)` — alternating delay pattern
- Other modes unknown (from [[Binary Structure]])

## Confirmed Maturity Flow
- CoreWriter actions enqueue with an explicit maturity timestamp and later execute from the delayed queue during `begin_block`.
- ABCI snapshots carry each queued entry's `matured_at` / `ma` directly; the runtime does not need to recompute maturity from volatility state when draining.
- Queue drain is based on `matured_at <= now_ms`.
- Expired delayed actions are dropped silently; CoreWriter entries currently use non-expiring `expires_after`.
- `enabled`, `delayer_mode`, `status_guard`, and `vty_trackers` are present in the state shape, but their exact control semantics are still not fully closed.

## Purpose
- CoreWriter actions from [[HyperEVM]] are "delayed onchain for a few seconds"
- Prevents frontrunning via lower-latency EVM path
- `vty_trackers` (volatility) may influence delay duration
- Delay scales with `delay_scale` parameter

## Strong RE Anchors Still Open
- `ActionDelayerStatus::Alternating { seconds }` is a confirmed binary enum variant.
- `ScaledAndRawVlmOnAlternating` strongly suggests a mode that combines scaled delay behavior with raw-volume tracking during alternating windows.
- `action_delayer_log_status` strongly suggests the binary has an explicit status-update/logging path separate from simple queue drain.
- `deterministic_vty_alert_n_bucket_samples` strongly suggests volatility alerting / bucket sampling feeds into the delayer control plane somewhere above plain `matured_at` checks.

## Links
- [[Exchange State]] — field 43 of the 57-field struct
- [[HyperEVM]] — CoreWriter actions hit the delayer
- [[Guards and Limits]] — status_guard is a BucketGuard

#security #latency #protection
