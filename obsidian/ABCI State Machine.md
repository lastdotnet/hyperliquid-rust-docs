# ABCI State Machine

The complete block processing pipeline for the Hyperliquid L1, from consensus to state commitment.

> **See also**: [[Block Lifecycle]] for the comprehensive hook-by-hook reference with every effect, guard interval, dispatch flow, and timing parameter.

## Block Processing Pipeline

```
HyperBFT Consensus
    │
    ▼
begin_block_logic()  -- CONFIRMED 9 effects (2026-04-02, VA 0x01e748e0)
    │  ── begin_block_logic_guard (reentrancy protection)
    │  ── _begin_block_to_commit timing
    │  ── 1. update_oracle
    │  ── 2. distribute_funding
    │  ── 3. apply_bole_liquidations
    │  ── 4. update_funding_rates
    │  ── 5. refresh_hip3_stale_mark_pxs (10s stale window)
    │  ── 6. prune_book_empty_user_states
    │  ── 7. update_staking_rewards
    │  ── 8. update_action_delayer      -- drain matured CoreWriter delayed actions
    │  ── 9. update_aligned_quote_token
    ▼
Process Actions (ordered)
    │  1. User actions (from broadcasters)
    │  2. System/governance actions (SetGlobal)
    │  3. Bridge actions (VoteEthDeposit, SignWithdrawal)
    │  4. EVM block production
    │  5. CoreWriter event processing
    │  6. L1 ↔ EVM transfer queue
    ▼
end_block()
    │  ── Mark price updates from book mids
    │  ── State hash update (LtHash16)
    │  ── State checkpointing
    ▼
VoteAppHash (validators agree on state)
```

Current local `main` now matches the widened 9-effect note: matured delayed
actions drain through the named `update_action_delayer` lane at slot 8. What
remains open is the ActionDelayer control-plane behavior around `enabled`,
`delayer_mode`, `status_guard`, and `vty_trackers`, not the existence of the
drain lane itself.

## Consensus Messages

From binary RE (`node/src/consensus/`):

| Type | Purpose |
|------|---------|
| **Vote** | Validator vote on a proposal |
| **Tx** | Transaction submission |
| **Propose** | Block proposal from leader (7 fields) |
| **Committed** | Finalized block with commit proof |
| **CommitProof** | 2/3+ stake committed (2+ fields) |

### SealedBlock (2 fields)
- `header`: SealedHeader (round + hash)
- `body`: BlockBody (actions + proposer + timestamp)

## Exchange State (57 fields)

The `Exchange` struct is the god object ("struct Exchange with 57 elements" in binary). Uses **full field names** in serde (NOT 2-char codes). See [[Exchange State]] for complete 57-field map. Key field categories:

### Core State
- `round`, `hardfork_version`, `blocks_processed`
- `scheduled_freeze_height`, `simulate_crash_height`

### Trading
- `perp_dexs[]` — per-asset order books
- `cancel_aggressive_orders_at_open_interest_cap_guard`
- `post_only_until_time`, `post_only_until_height`
- `spot_disabled`, `hyperliquidity`

### User State
- `user_to_display_name`, `user_to_scheduled_cancel`
- `user_to_evm_state`, `user_to_evm_to_l1_wei_remaining`
- `dex_abstraction_enabled`

### Trackers
- `spot_twap_tracker`, `multi_sig_tracker`
- `staking_link_tracker`, `validator_l1_vote_tracker`
- `validator_l1_stream_tracker`, `app_hash_vote_tracker`
- `action_delayer`, `validator_power_updates`

### Guards (BucketGuard pattern — 16 instances)
- `begin_block_logic_guard` — reentrancy
- `partial_liquidation_cooldown` — 30s between liquidations
- `hip3_stale_mark_px_guard` — HIP-3 price staleness
- `book_empty_user_states_pruning_guard`
- `prune_agent_idx`

### EVM
- `hyper_evm` (13 sub-fields), `evm_enabled`
- `disabled_precompiles`, `initial_usdc_evm_system_balance`

### HIP-3
- `default_hip3_limits`, `hip3_stale_mark_px_guard`, `hip3_no_cross`

### Gas Auctions
- `register_token_gas_auction`, `perp_deploy_gas_auction`
- `spot_pair_deploy_gas_auction`

## Gossip State Sync

When a node connects behind the network:
1. Leader sends full ABCI state snapshot ("sending abci_state")
2. Leader sends EVM key-value store ("sending evm kvs")
3. Connection drops and reconnects for streaming

### Connection Flow
```
TCP connect → send greeting → recv greeting
    → performing checks → verify_rpc → begin streaming
```

### Rejection Reasons
- "max peers reached"
- "not validator or sentry"
- "abci state request rate limited"
- "no quorum yet"
- "failed to verify peer rpc"
- "peer is already connected"

### Health States
- **fresh** — up to date
- **stale** — falling behind
- **missing** — not connected

## Builder Fees

1. User sends `ApproveBuilderFee` (builder, max_fee, chain, nonce, expiry)
2. Builder includes user txs up to max_fee
3. Fees collected in `collected_builder_fees`
4. `trials_disabled` — fee trials can be disabled

## Our Implementation Status (updated 2026-04-02)

| Component | Status | Notes |
|-----------|--------|-------|
| begin_block | Partial | Repo has a widened begin-block / execution-state surface. The delayed-action lane is confirmed, but its exact placement relative to the 9-effect body is still open. |
| Action dispatch (80 types) | Partial | All variants parsed; **64 of ~80 return Unimplemented** — only ~16 have real handlers |
| Order matching | Complete | BTreeMap + VecDeque + O(1) OID cancel |
| Clearinghouse | Partial | Positions, margin basics; liquidation, isolated margin, ADL NOT implemented |
| Funding settlement | Partial | 1h funding path in `begin_block`; multipliers, interest rates, Binance formula path not wired |
| end_block | Partial | Mark-price update path wired; response-hash uses placeholder tuples, not cracked structs |
| State hash | Partial | LtHash16 algorithm cracked + implemented; response struct field orders cracked but NOT wired |
| EVM integration | Partial | revm 36, precompiles read-only, CoreWriter decode is a stub, dual-block not implemented |
| Bridge | Mock | Auto-confirm deposits/withdrawals; real bridge actions unimplemented |
| Oracle | Mock | Static/Synthetic/Live modes; validator vote aggregation not implemented |
| Broadcaster | Mock | FIFO queue with bundle assembly |
| Bootstrap | Partial | 4 of 57 Exchange fields parsed from .rmp; 53 skipped |

## Links
- [[Exchange State]] — 57-field structure
- [[HyperBFT]] — consensus protocol
- [[App Hash]] — state hash voting
- [[Matching Engine]] — order processing
- [[HyperEVM]] — EVM block production
- [[Devnet]] — local development network

#state-machine #abci #consensus #architecture
