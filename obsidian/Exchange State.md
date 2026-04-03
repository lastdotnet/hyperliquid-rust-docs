# Exchange State

The central state object of [[Hyperliquid]] — **57 fields** (binary says "struct Exchange with 57 elements") that contain ALL L1 state. Every block's `begin_block -> process actions -> end_block` mutates this struct.

**CONFIRMED 2026-04-02**: Complete field map extracted from ABCI state snapshot (942770000.rmp, 1.1GB) cross-referenced against binary rodata (8 identical serializer string tables). Exchange uses **full field names** in serde (NOT 2-char rename codes). The Locus sub-struct uses 3-char abbreviated names.

**UPDATED 2026-04-03**: New testnet binary (sha256 `1a99c892...`, built `2026-04-03 10:43:39 UTC`, commit `331bef9b`) confirms all 57 fields. Field 54 is `hilo` (not `hil`), field 56 is `hpt`. Full serde struct chain at `.rodata` offset `0x543428` includes complete field orders for Bridge2, Context, Staking, FundingTracker, Clearinghouse, and all nested types. See `docs/generated/re/runs/2026-04-03-struct-chain.md` for the complete parsed chain.

## Complete Exchange Field Map (57 fields)

Verified from binary rodata at 0x386138 (and 7 duplicate copies). Field names are concatenated in exact serialization order. Binary descriptor: `struct Exchange with 57 elements`.

| Idx | Field | Type | Value/Detail |
|-----|-------|------|--------------|
| 0 | `locus` | [[Locus]](15 keys) | Core state container (cls, scl, ctx, ftr, ust, chn, pdl, uaR, blp, qus, ctr, uac, vlt, hcm, aux) |
| 1 | `perp_dexs` | Vec\<PerpDex\>(9) | Perpetual DEX instances; each has {books, funding_tracker, twap_tracker} |
| 2 | `spot_books` | Vec\<SpotBook\>(307) | Spot order books; each has {halfs, oid_to_key, cloid_tracker, book_orders, asset, user_states, last_trade_px, total_sz} |
| 3 | `agent_tracker` | AgentTracker(5) | {user_to_main_agent, user_to_extra_agents, agent_to_user, address_to_nonce_set, user_to_pending_agent_removals} |
| 4 | `funding_distribute_guard` | BucketGuard | {last_bucket, bucket_millis} |
| 5 | `funding_update_guard` | BucketGuard | {last_bucket, bucket_millis} |
| 6 | `sub_account_tracker` | SubAccountTracker(2) | {user_to_sub_accounts, sub_account_to_master} |
| 7 | `allowed_liquidators` | Vec\<Address\>(5) | Liquidator whitelist (5 addresses on mainnet) |
| 8 | `bridge2` | Bridge2(10) | [[Bridge]] v2: {eth_id_to_deposit_votes, finished_deposits_data, withdrawal_signatures, withdrawal_finalized_votes, finished_withdrawal_to_time, validator_set_signatures, validator_set_finalized_votes, bal, last_pruned_deposit_block_number, oaw} |
| 9 | `staking` | Staking(6) | {epoch_states, active_epoch, cur_epoch_state, cur_epoch, epoch_duration_seconds, allowed_validators} |
| 10 | `c_staking` | CStaking(23) | Full consensus staking: {stakes, allowed_validators, allow_all_validators, broadcasters, validator_to_profile, validator_to_last_signer_change_time, delegations, proposals, stakes_decay_bucket_guard, jailed_signers, jail_vote_tracker, jail_until, signer_to_last_manual_unjail_time, native_token_reserve, stage_reward_bucket_guard, validator_to_staged_rewards, distribute_reward_bucket_guard, user_states, pending_withdrawals, self_delegation_requirement, disabled_validators, disabled_node_ips, validator_to_state} |
| 11 | `funding_err_dur_guard` | ErrDurGuard(3) | {err_duration, first_err_time, last_err_time} |
| 12 | `max_order_distance_from_anchor` | f64 | 0.95 (price band: orders must be within 95% of anchor) |
| 13 | `scheduled_freeze_height` | Option\<u64\> | null normally; set to freeze chain at height for upgrades |
| 14 | `simulate_crash_height` | Option\<u64\> | null normally; testing crash simulation |
| 15 | `validator_power_updates` | HashMap\<PubKey, u64\>(4) | Pending validator power changes |
| 16 | `cancel_aggressive_orders_at_open_interest_cap_guard` | BucketGuard | {last_bucket, bucket_millis} |
| 17 | `last_hlp_cancel_time` | DateTime\<Utc\> | Timestamp of last HLP cancel |
| 18 | `post_only_until_time` | DateTime\<Utc\> | Post-only mode end time |
| 19 | `post_only_until_height` | u64 | Post-only mode end height (e.g. 930593206) |
| 20 | `spot_twap_tracker` | SpotTwapTracker(2) | {running_heap, user_to_id_to_state} |
| 21 | `user_to_display_name` | HashMap\<Address, String\>(2160) | User display names |
| 22 | `book_empty_user_states_pruning_guard` | BucketGuard | {last_bucket, bucket_millis} |
| 23 | `user_to_scheduled_cancel` | HashMap\<Address, ScheduledCancel\>(144) | Scheduled order cancellations |
| 24 | `hyperliquidity` | Hyperliquidity(2) | {spot_to_range, ensure_order_bucket_guard} |
| 25 | `spot_disabled` | bool | false on mainnet |
| 26 | `prune_agent_idx` | u64 | Agent pruning index (e.g. 5) |
| 27 | `max_hlp_withdraw_fraction_per_day` | f64 | 0.1 (10% daily HLP withdrawal limit) |
| 28 | `hlp_start_of_day_account_value` | u64 | HLP account value at day start (raw integer, e.g. 448364411313797) |
| 29 | `register_token_gas_auction` | GasAuction(4) | {bucket_guard, start_gas_wei, end_gas_wei, token} |
| 30 | `perp_deploy_gas_auction` | GasAuction(4) | {bucket_guard, start_gas_wei, end_gas_wei, token} |
| 31 | `spot_pair_deploy_gas_auction` | GasAuction(4) | {bucket_guard, start_gas_wei, end_gas_wei, token} |
| 32 | `hyper_evm` | HyperEvm(13) | [[HyperEVM]]: {state2, latest_block2, small_block_context, big_block_context, is_initialized, l1_to_evm_native_token_queue, l1_to_evm_erc20_queue, l1_transfers_enabled, disabled_core_writer_actions, put, ul, ub, ft} |
| 33 | `vtg` | RawSignatureTracker(3) | Validator tracking guard: {inner (empty list), name="vtg", assert_unique_values=false} |
| 34 | `app_hash_vote_tracker` | AppHashVoteTracker(3) | {tracker, quorum_app_hashes, highest_quorum_height} |
| 35 | `begin_block_logic_guard` | BucketGuard | {last_bucket, bucket_millis} |
| 36 | `user_to_evm_state` | HashMap\<Address, EvmState\>(778843) | Per-user EVM state mappings |
| 37 | `multi_sig_tracker` | MultiSigTracker(1) | {user_to_multi_sig_signers} |
| 38 | `reserve_accumulators` | Vec\<(u32, T)\>(4) | Reserve tracking per token (keys: 107, 207, 232, 255) |
| 39 | `partial_liquidation_cooldown` | f64 | 30.0 seconds between liquidations |
| 40 | `evm_enabled` | bool | true on mainnet |
| 41 | `validator_l1_vote_tracker` | ValidatorL1VoteTracker(3) | {prune_bucket_guard, action_to_tracker, active} |
| 42 | `staking_link_tracker` | StakingLinkTracker(1) | {user_to_staking_link2} |
| 43 | `action_delayer` | [[Action Delayer]](9) | {delayed_actions, user_to_n_delayed_actions, n_total_delayed_actions, max_n_delayed_actions, vty_trackers, status_guard, enabled, delayer_mode, delay_scale} |
| 44 | `default_hip3_limits` | Hip3Limits(6) | {total_oi_cap2, oi_sz_cap_per_perp, max_transfer_ntl2, max_under_initial_margin_transfer_ntl2, daily_px_range, max_n_users_with_positions} |
| 45 | `hip3_stale_mark_px_guard` | BucketGuard | {last_bucket, bucket_millis} |
| 46 | `disabled_precompiles` | Vec\<u32\> | Empty list on mainnet |
| 47 | `lvt` | DateTime\<Utc\> | Last validator time (e.g. "2026-04-01T21:37:58.542650736") |
| 48 | `hip3_no_cross` | bool | false on mainnet |
| 49 | `initial_usdc_evm_system_balance` | u64 | i64::MAX (9223372036854775807) — sentinel for "unlimited" |
| 50 | `validator_l1_stream_tracker` | ValidatorL1StreamTracker(1) | {risk_free_rate} |
| 51 | `last_aligned_quote_token_sample_time` | DateTime\<Utc\> | Quote token sampling timestamp |
| 52 | `user_to_evm_to_l1_wei_remaining` | HashMap\<Address, T\> | EVM-to-L1 transfer remaining balances (empty on mainnet) |
| 53 | `dex_abstraction_enabled` | bool | true on mainnet |
| 54 | `hilo` | HyperliquidityImpactLimiter(3) | {m="e" (mode), half_life=600.0, max_slippage=0.02} |
| 55 | `oh` | Vec\<(DateTime, OracleSnapshot)\>(100) | Oracle history: ring buffer of 100 snapshots, each with {pxs, external_perp_pxs, err_dur_guard, l} |
| 56 | `hpt` | AlignedQuoteTokenTracker | **DISCOVERED**: Aligned quote token state tracker. Per-token: isAligned, firstAlignedTime, evmMintedSupply, dailyAmountOwed, predictedRate. Validators submit SOFR-based risk-free rates daily via `validatorL1Stream` (aqa-publisher sidecar). Protocol computes stake-weighted median. Related: field [51] `last_aligned_quote_token_sample_time`, FeeSchedule field [6] `aligned_quote_token_config`. Info API: `alignedQuoteTokenInfo`. |

## Locus Sub-Fields (15 keys, binary says 14 + aux added later)

| Idx | Key | Full Name | Type | Detail |
|-----|-----|-----------|------|--------|
| 0 | `cls` | clearinghouses | Vec\<Clearinghouse\>(9) | Each has 18 fields: {meta(PerpMeta/6), user_states(UserStates/6), oracle, total_net_deposit, total_non_bridge_deposit, perform_auto_deleveraging, adl_shortfall_remaining, bridge2_withdraw_fee, daily_exchange_scaled_and_raw_vlms, halted_assets, override_max_signed_distances_from_oracle, max_withdraw_leverage, last_set_global_time, usdc_ntl_scale(f64,u64), isolated_external, isolated_oracle, moh(MainOrHip3 enum), znfn(u64 fill nonce)} |
| 1 | `scl` | spot_clearinghouse | SpotClearinghouse(12) | {meta, user_states, daily_exchange_scaled_and_raw_vlms, token_to_genesis, token_to_max_supply, token_fees_collected, native_token, spot_dusting_opted_out_users, evm_transfer_escrows, token_to_pending_evm_contract, pmr, soh} |
| 2 | `ctx` | context | Context(12) | {initial_height, hardfork, height, tx_index, round, time, next_oid, next_lid, next_twap_id, system_nonce, use_system_nonce, mac} |
| 3 | `ftr` | fee_tracker | FeeTracker(10) | {fee_schedule, unrewarded_users, user_states, referrer_states, code_to_referrer, referral_bucket_millis, total_fees_collected, total_spot_fees_collected, collected_builder_fees, trials_disabled} |
| 4 | `ust` | user_states | HashMap\<Address, UserState\>(1.8M) | All user balance/position state |
| 5 | `chn` | chain | String | "Mainnet" / "Testnet" / "Sandbox" / "Local" |
| 6 | `pdl` | perps_disabled_launch | bool | false on mainnet |
| 7 | `uaR` | user_action_registry | Vec<?> | Empty on mainnet (capital R notable) |
| 8 | `blp` | bole_pool | BolePool(19) | BOLE lending pool: {dp, du, o, pmmt, e, pmv, r, p, n, b, bog, pmdl, pml, uobc, ubgl, ubul, bwo, mbs, u} |
| 9 | `qus` | queue_signers | Vec\<Address\>(1) | Queue signer addresses |
| 10 | `ctr` | counters | Counters(7) | {tn, vn, tc, sc, to, so, ac} -- transaction/validation/system counters |
| 11 | `uac` | user_account_configs | HashMap\<Address, T\>(30833) | Per-user account configuration |
| 12 | `vlt` | vaults | HashMap\<Address, Vault\>(9376) | All vault state (Vault with 13 fields) |
| 13 | `hcm` | hot_cold_mode | String | "enabled" on mainnet |
| 14 | `aux` | auxiliary | Vec<?> | Empty on mainnet (added after binary version, not in binary's 14-field struct) |

## Key Type Clarifications

**NOT 2-char rename codes**: Exchange uses **full field names** in serde serialization. The Locus sub-struct uses 3-char abbreviated names (cls, scl, ctx, etc.). The user-facing 2-char codes (ct, cl, pd, st, cs) were a misconception -- they don't appear anywhere in the binary's 8 serializer string tables.

**Field name encoding**: ABCI state snapshots (.rmp) use named-key MessagePack. The wire/gossip format uses custom bincode-fork (positional, no field names). The 8 identical copies of the field name table in the binary are from serde's derive macro (Serialize + Deserialize visitors).

## BucketGuard Pattern (7 instances)
All BucketGuards share the same 2-field structure: `{last_bucket, bucket_millis}`.
Found at Exchange fields: 4, 5, 16, 22, 35, 45, and inside action_delayer.status_guard.

## State Serialization
- **Snapshots**: MessagePack (named keys), saved every ~20 min as `.rmp`
- **Wire**: Custom bincode-fork (positional), streamed via [[Gossip Protocol]]
- **JSON**: Via `hl-node translate-abci-state` (5.5GB for full state)
- **State hash**: [[LtHash]] (BLAKE3 XOF, incremental)

## State Size (from snapshot 942770000, 2026-04-01)
- 1,823,730 user states (ust)
- 1,224,601 spot clearinghouse user states
- 778,843 EVM user states
- 9,376 vaults
- 9 perp DEX instances, 307 spot books
- 100 oracle history snapshots
- 1.1GB as .rmp

## Links
- [[Locus]] -- core position/clearinghouse container
- [[Clearinghouse]] -- 18-field position/margin state
- [[Matching Engine]] -- order books
- [[Action Types]] -- 80 action types that mutate this state
- [[LtHash]] -- how state is hashed for consensus
- [[Guards and Limits]] -- 7 BucketGuards in this struct

#state #exchange #architecture
