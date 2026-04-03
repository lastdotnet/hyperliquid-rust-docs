# Yellow Paper — Protocol Specification

The protocol-truth reference for the Hyperliquid L1 reimplementation. Every claim here is tagged with its evidence level.

**Evidence levels**: `CONFIRMED` = verified from binary RE or official docs. `OBSERVED` = seen in runtime/snapshots. `IMPLEMENTED` = coded in hlx. `INFERRED` = deduced from behavior.

Use the [White Paper](../whitepaper/index.md) for narrative architecture. Use the [Hypurrliquid Paper](../paper/index.html) for the full RE reference with Mermaid diagrams.

---

## 1. Block Lifecycle (46 Hook Points)

`CONFIRMED` from binary VA 0x01e748e0 and string at 0x539f8a.

### 1.1 Pipeline

```
RecoverUsers → BeginBlock (9+5 effects) → DeliverSignedActions (97 types)
→ ProcessFills → EVM Block → EndBlock → Commit → VoteAppHash
```

### 1.2 BeginBlock — 9 Named Effects

| # | Effect | Guard Interval | Evidence |
|---|--------|---------------|----------|
| 1 | `update_oracle` | — | `CONFIRMED` |
| 2 | `distribute_funding` | 8000ms | `CONFIRMED` |
| 3 | `apply_bole_liquidations` | — | `CONFIRMED` |
| 4 | `update_funding_rates` | 8000ms | `CONFIRMED` |
| 5 | `refresh_hip3_stale_mark_pxs` | from snapshot | `CONFIRMED` |
| 6 | `prune_book_empty_user_states` | 60000ms | `CONFIRMED` |
| 7 | `update_staking_rewards` | — | `CONFIRMED` |
| 8 | `update_action_delayer` | — | `CONFIRMED` |
| 9 | `update_aligned_quote_token` | — | `CONFIRMED` |

### 1.3 Supplementary Effects

| Effect | Guard | Evidence |
|--------|-------|----------|
| `check_trigger_orders` | — | `IMPLEMENTED` |
| `cancel_aggressive_orders_at_oi_cap` | 1000ms | `CONFIRMED` |
| `validator_l1_vote_tracker_prune_expired` | 60000ms | `CONFIRMED` |
| `reset_recent_ois` | 1000ms | `CONFIRMED` |
| `update_stale_mark_guards` | per-book | `CONFIRMED` |

### 1.4 EndBlock

Mark prices updated from book mid-prices: `mark_prices[asset] = book.mid_price()`.

### 1.5 VoteAppHash

Every 2000 blocks. Quorum at 2/3+ stake. Hash: `rmp_serde → blake3 XOF (2048B) → paddw u16 → SHA-256 (32B)`.

---

## 2. Exchange State (57 Fields)

`CONFIRMED` from serde struct chain at `.rodata` offset `0x543428` in testnet binary `331bef9b` (2026-04-03).

| # | Field | Type | Parsed | Evidence |
|---|-------|------|--------|----------|
| 0 | `locus` | Locus(15) | Yes | `CONFIRMED` |
| 1 | `perp_dexs` | Vec\<PerpDex\> | Yes | `CONFIRMED` |
| 2 | `spot_books` | Vec\<SpotBook\> | Yes | `CONFIRMED` |
| 3 | `agent_tracker` | AgentTracker(5) | Yes | `CONFIRMED` |
| 4 | `funding_distribute_guard` | BucketGuard | Yes | `CONFIRMED` |
| 5 | `funding_update_guard` | BucketGuard | Yes | `CONFIRMED` |
| 6 | `sub_account_tracker` | SubAccountTracker(2) | Yes | `CONFIRMED` |
| 7 | `allowed_liquidators` | Vec\<Address\> | Yes | `CONFIRMED` |
| 8 | `bridge2` | Bridge2(10) | Yes | `CONFIRMED` |
| 9 | `staking` | Staking(6) | Yes | `CONFIRMED` |
| 10 | `c_staking` | CStaking(23) | Yes | `CONFIRMED` |
| 11 | `funding_err_dur_guard` | BucketGuard | Yes | `CONFIRMED` |
| 12 | `max_order_distance_from_anchor` | f64 | Yes | `CONFIRMED` |
| 13 | `scheduled_freeze_height` | u64 | Yes | `CONFIRMED` |
| 14 | `simulate_crash_height` | u64 | Yes | `CONFIRMED` |
| 15 | `validator_power_updates` | Vec | Skip | `CONFIRMED` |
| 16 | `cancel_oi_cap_guard` | BucketGuard | Yes | `CONFIRMED` |
| 17 | `last_hlp_cancel_time` | f64 | Yes | `CONFIRMED` |
| 18 | `post_only_until_time` | f64 | Yes | `CONFIRMED` |
| 19 | `post_only_until_height` | u64 | Yes | `CONFIRMED` |
| 20 | `spot_twap_tracker` | JSON | Yes | `CONFIRMED` |
| 21 | `user_to_display_name` | HashMap | Yes | `CONFIRMED` |
| 22 | `book_empty_user_states_pruning_guard` | BucketGuard | Yes | `CONFIRMED` |
| 23 | `user_to_scheduled_cancel` | HashMap | Yes | `CONFIRMED` |
| 24 | `hyperliquidity` | 2 fields | Yes | `CONFIRMED` |
| 25 | `spot_disabled` | bool | Yes | `CONFIRMED` |
| 26 | `prune_agent_idx` | u64 | Yes | `CONFIRMED` |
| 27 | `max_hlp_withdraw_fraction_per_day` | f64 | Yes | `CONFIRMED` |
| 28 | `hlp_start_of_day_account_value` | f64 | Yes | `CONFIRMED` |
| 29 | `register_token_gas_auction` | GasAuction(4) | Yes | `CONFIRMED` |
| 30 | `perp_deploy_gas_auction` | GasAuction(4) | Yes | `CONFIRMED` |
| 31 | `spot_pair_deploy_gas_auction` | GasAuction(4) | Yes | `CONFIRMED` |
| 32 | `hyper_evm` | HyperEvmState(13) | Yes | `CONFIRMED` |
| 33 | `vtg` | JSON | Yes | `CONFIRMED` |
| 34 | `app_hash_vote_tracker` | 3 fields | Yes | `CONFIRMED` |
| 35 | `begin_block_logic_guard` | BucketGuard | Yes | `CONFIRMED` |
| 36 | `user_to_evm_state` | HashMap (778K) | Skip | `CONFIRMED` |
| 37 | `multi_sig_tracker` | 1 field | Yes | `CONFIRMED` |
| 38 | `reserve_accumulators` | JSON | Yes | `CONFIRMED` |
| 39 | `partial_liquidation_cooldown` | f64 (30.0) | Yes | `CONFIRMED` |
| 40 | `evm_enabled` | bool | Yes | `CONFIRMED` |
| 41 | `validator_l1_vote_tracker` | 3 fields | Yes | `CONFIRMED` |
| 42 | `staking_link_tracker` | JSON | Yes | `CONFIRMED` |
| 43 | `action_delayer` | ActionDelayer(9) | Yes | `CONFIRMED` |
| 44 | `default_hip3_limits` | JSON | Yes | `CONFIRMED` |
| 45 | `hip3_stale_mark_px_guard` | BucketGuard | Yes | `CONFIRMED` |
| 46 | `disabled_precompiles` | Vec | Yes | `CONFIRMED` |
| 47 | `lv` | JSON | Yes | `CONFIRMED` |
| 48 | `hip3_no_cross` | bool | Yes | `CONFIRMED` |
| 49 | `initial_usdc_evm_system_balance` | f64 | Yes | `CONFIRMED` |
| 50 | `validator_l1_stream_tracker` | JSON | Yes | `CONFIRMED` |
| 51 | `last_aligned_quote_token_sample_time` | f64 | Yes | `CONFIRMED` |
| 52 | `user_to_evm_to_l1_wei_remaining` | HashMap | Yes | `CONFIRMED` |
| 53 | `dex_abstraction_enabled` | bool | Yes | `CONFIRMED` |
| 54 | `hilo` | HyperliquidityImpactLimiter(3) | Yes | `CONFIRMED` |
| 55 | `oh` | Vec\<OracleSnapshot\>(100) | Yes | `CONFIRMED` |
| 56 | `hpt` | AlignedQuoteTokenTracker | Yes | `DISCOVERED` |

**57/57 fields declared. 55/57 deeply parsed from snapshots. 2 skipped (user_to_evm_state, validator_power_updates).**

---

## 3. Action Surface (97 Variants, 126 Sub-Types)

`CONFIRMED` from binary dispatcher at VA 0x0272a320. 90 mainnet + 7 testnet.

| Category | Count | Key Types |
|----------|-------|-----------|
| Trading | 9 | order, cancel, cancelByCloid, modify, batchModify, TWAP, liquidate |
| Margin | 3 | updateLeverage, updateIsolatedMargin, topUpIsolatedOnlyMargin |
| Transfers | 6 | usdSend, spotSend, sendAsset, withdraw3, sendToEvmWithData |
| Sub-Accounts | 3 | createSubAccount, subAccountTransfer, subAccountSpotTransfer |
| Vaults | 6 | createVault, vaultTransfer, vaultDistribute, vaultModify, netChildVaultPositions, setVaultDisplayName |
| Staking | 5 | tokenDelegate, claimRewards, linkStakingUser, registerValidator, extendLongTermStaking |
| Account | 7 | approveAgent, setDisplayName, setReferrer, convertToMultiSigUser, approveBuilderFee, startFeeTrial, userPortfolioMargin |
| DeFi | 5 | borrowLend, userDexAbstraction, userSetAbstraction, agentSetAbstraction, agentEnableDexAbstraction |
| Governance | 3 | govPropose, govVote, voteGlobal (22 sub-types) |
| HIP-3/4 | 3 | SetGlobal (6 sub), perpDeploy (8 sub), spotDeploy |
| Validator | 9 | CValidator, CSigner, validatorL1Vote, validatorL1Stream, bridge votes (5) |
| System | 11 | SystemBole, SystemSpotSend, CWithdraw, CUserModify, EvmUserModify, etc. |
| Special | 7 | evmRawTx, multiSig, noop, VoteAppHash, ForceIncreaseEpoch, userOutcome (4 sub), settleFraction |
| Additional | 8 | priorityBid, registerSpot, disableDex, cancelAllOrders, quarantineUser, etc. |

**All 97 variants implemented. Zero stubs.**

---

## 4. Funding Rate

`CONFIRMED` from binary assertions.

- **EMA**: DeterministicEma with fraction-based accumulation (num/denom). `assert!(decay <= 1.)`, `assert!(self.denom > 0.)`.
- **Funding payment**: `payment = szi × oracle_px × (cum_funding_global - pos.cum_funding_entry)`
- **Direction**: Positive rate = longs pay shorts.
- **Guard interval**: 8000ms for both distribute and update.
- **Per-asset multiplier**: Set via `SetFundingMultiplier` VoteGlobal.

---

## 5. Liquidation & ADL

`CONFIRMED` from binary constants and docs.

- **Margin check**: `margin_available = account_value - initial_margin_required`. Liquidatable when < 0.
- **Isolated**: Per-position check with allocated margin.
- **BOLE liquidation threshold**: Health factor < 0.8.
- **BOLE cooldown**: 30 seconds for partial liquidations.
- **BOLE types**: Partial (50% seize), Market (full repay), Backstop (default liquidator).
- **ADL**: Triggered when shortfall persists after liquidation. Counterparties ranked by ROE (highest first). Position reduced proportionally.
- **ADL priority**: `ROE = unrealized_pnl / margin_allocated`.

---

## 6. State Hashing

`CONFIRMED` algorithm: `rmp_serde → blake3 XOF (2048B) → paddw u16 (LtHash16) → SHA-256 (32B)`.

- **14 L1 categories** + **3 EVM categories** = 17 accumulators.
- **L1 accumulators**: Running sums from genesis, stored in BSS (NOT rebuilt from state).
- **EVM accumulators**: accounts_hash, contracts_hash, storage_hash.
- **Heartbeat `concise_lt_hashes`**: EVM only. L1 accumulators NOT in heartbeats.

---

## 7. CoreWriter (EVM → L1)

`CONFIRMED` from binary and official docs.

- **Precompile address**: `0x3333333333333333333333333333333333333333`
- **Function**: `sendRawAction(bytes)` selector `0x17938e13`
- **Encoding**: `abi.encodePacked(uint8(1), uint24(actionId), abi.encode(params...))`
- **15 action types**: LimitOrder, VaultTransfer, TokenDelegate, StakingDeposit, StakingWithdraw, SpotSend, UsdClassTransfer, FinalizeEvmContract, AddApiWallet, CancelByOid, CancelByCloid, ApproveBuilderFee, SendAsset, ReflectEvmSupplyChange, BorrowLend.
- **Delay**: `delay_scale × 3000ms` (configurable). Actions fire in next block's begin_block Effect 8.
- **In replay mode**: Raw tx calldata scanned via RLP parser without EVM execution.

---

## 8. Aligned Quote Token (HPT)

`DISCOVERED` from Hyperliquid docs + API.

- **Exchange field [56]** `hpt` = AlignedQuoteTokenTracker.
- **Per-token state**: isAligned, firstAlignedTime, evmMintedSupply, dailyAmountOwed, predictedRate.
- **Validator sidecar**: `aqa-publisher` ([github.com/native-markets/aqa-publisher](https://github.com/native-markets/aqa-publisher)).
- **Rate source**: 30-day SOFR average from NY Fed, FRED, OFR (median of 3 sources).
- **Submission**: `validatorL1Stream` action with `riskFreeRate` field, daily at 22:00 UTC.
- **Protocol rate**: Stake-weighted median of all active validator votes.
- **Fee benefits**: 20% lower taker, 50% better maker rebate for aligned stablecoins.

---

## 9. Gossip Protocol

`CONFIRMED` from binary RE.

- **Greeting**: 6-byte bincode-fork `TcpGreeting` enum. Chain: Local=0, Sandbox=1, Testnet=2, Mainnet=3.
- **Frame format**: `[u32 BE len][u8 kind][body]`, often LZ4 compressed.
- **Connection**: Server-side `connection_checks` — unknown peers rejected after 5s timeout.
- **Loopback**: Dominated by `MsgConcise`/`OutMsgConcise` (tag 27/28).

---

## 10. Guard System

All rate-limited effects use `BucketGuard`:

```
struct BucketGuard {
    last_time: Option<u64>,   // ms
    interval_ms: u64,
}
```

| Guard | Interval | Effect |
|-------|----------|--------|
| `funding_distribute_guard` | 8000ms | Funding settlement |
| `funding_update_guard` | 8000ms | Premium sampling |
| `cancel_oi_cap_guard` | 1000ms | OI cap order cancellation |
| `book_empty_user_states_pruning_guard` | 60000ms | Book cleanup |
| `reset_recent_oi_guard` | 1000ms | HIP-3 OI reset |
| `hip3_stale_mark_px_guard` | From snapshot | Stale mark refresh |
| `begin_block_logic_guard` | 1000ms | Reentrancy protection |
| `funding_err_dur_guard` | 60000ms | Funding error duration |

---

## 11. Implementation Status

| Component | Status | Tests |
|-----------|--------|-------|
| Action handlers (97) | All implemented | 330+ |
| Begin-block (9 + 5) | All implemented | Tested |
| Bootstrap (57 fields) | 55/57 parsed | Bootstrap test |
| CoreWriter decoder | 15 actions + RLP scanner | 7 tests |
| ADL + BOLE liquidation | Execution implemented | ADL plan + candidates tests |
| Pipeline stages | 6 stages with real logic | 4 tests |
| RPC (30+ endpoints) | Consolidated in hl-rpc | Info tests |
| Integration tests | 18 E2E devnet tests | All pass |
| **Total** | **482 tests, 0 failures** | **Release: 11MB** |
