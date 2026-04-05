# Yellow Paper — Protocol Specification

The protocol-truth reference for the Hyperliquid L1 reimplementation. Every claim here is tagged with its evidence level.

**Evidence levels**: `CONFIRMED` = verified from binary RE or official docs. `OBSERVED` = seen in runtime/snapshots. `IMPLEMENTED` = coded in hlx. `INFERRED` = deduced from behavior.

Use the [White Paper](../whitepaper/index.md) for narrative architecture. Use the [Hypurrliquid Paper](../paper/index.html) for the full RE reference with Mermaid diagrams. Use the [Action Inventory](./action-inventory.md) for the current action-family and sub-variant tables. Use the [Truth Register](../findings/truth-register.md) when you want the compact fact ledger instead of the full chapter narrative. For the denser subsystem splits, use [Hashing](../hashing/index.html), [Staking & Validators](../staking/index.html), [Bridge2](../bridge/index.html), and [Outcomes](../outcomes/index.html).

---

## 0. Truth Surfaces

This paper is the technical narrative, not the only truth surface. Read it together with:

- [Truth Register](../findings/truth-register.md) for the compact confirmed/active fact ledger
- [Open Claims](../status/open-claims.md) for unresolved work
- [Protocol Scope Matrix](./protocol-scope-matrix.md) for `protocol` vs `testnet_impl` vs `local_impl`
- [Action Inventory](./action-inventory.md) for action-family and sub-variant tables

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

Current local `main` matches the widened 9-effect note on this ordering. What remains open on ActionDelayer is control semantics (`enabled`, `delayer_mode`, `status_guard`, `vty_trackers`), not the named drain slot.

### 1.4 EndBlock

Mark prices updated from book mid-prices: `mark_prices[asset] = book.mid_price()`.

### 1.5 VoteAppHash

Every 2000 blocks. Quorum at 2/3+ stake. Final `app_hash` is `first_16(SHA-256(L1_combined)) || first_16(SHA-256(EVM_combined))`, where each half comes from the `rmp_serde → blake3 XOF (2048B) → paddw u16` LtHash pipeline rather than one flat 14-category digest.

### 1.6 Consensus / Topology Surface

Current confirmed testnet-facing networking and consensus facts:

- ingress is concentrated through a broadcaster layer
- proposer rotation uses stake-weighted `RoundRobinTtl`
- QC and TC certificates drive the two-chain commit rule
- validator/sentry admission checks gate the transport layer
- ports `4001` and `4002` split block-streaming and peer/RPC verification traffic

Treat the exact operator deployment shape as still chain-scoped, but keep these surfaces aligned across lifecycle, gossip, and consensus notes.

### 1.7 Validator Lifecycle / Jail Surface

Current confirmed validator-control facts:

- epochs are time-based via `epoch_duration_seconds`
- active validators are recomputed at epoch boundaries
- signer admission checks search `epoch_states[cur_epoch]`
- jailed signers are excluded from normal proposer / signing flow
- `CSignerAction` currently has `unjailSelf`, `jailSelf`, and `jailSelfIsolated`
- manual unjail is rate-limited by `signer_to_last_manual_unjail_time`

What remains open on this lane is threshold/trigger detail, not the existence of
the lifecycle surface.

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

Current testnet notes widen the action surface to 97 variants. The local engine has broad typed coverage, but parity work remains active and this page should not be read as a blanket “zero gaps” claim.

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

- **11 L1 categories** + **3 EVM categories** = 14 accumulators.
- **L1 accumulators**: Running sums from genesis maintained in live state, not the stale BSS cache that earlier notes treated as canonical.
- **EVM accumulators**: accounts_hash, contracts_hash, storage_hash.
- **Heartbeat `concise_lt_hashes`**: EVM only. The traced heartbeat surface does not carry the full L1 accumulator state.

### 6.1 AppHash Split

Current repo truth is that the final `app_hash` is `first_16(SHA-256(L1_combined)) || first_16(SHA-256(EVM_combined))`, not a single SHA-256 over one merged 14-accumulator buffer.

### 6.2 RespHash Backend Split

Current repo truth is intentionally narrower than "hashing is solved":

- the code explicitly tracks `Mainnet` versus `Testnet` RespHash backends
- the backend split is real and should remain chain-scoped in docs and replay work
- the local implementation does **not** yet claim full mainnet serializer parity across all response families

Treat `RespHashMode::Mainnet` as backend selection, not as proof that every
mainnet response serializer is already closed.

### 6.3 Current Serializer Facts

Treat the following as closed facts, not loose hypotheses:

- **G-family success/error split**: hybrid actions do **not** hash a 3-field payload on success. Success is the same 1-field `{status:"success"}` map used by the H-family status serializer. The 3-field `{status:"err", success:false, error:<msg>}` payload is error-only.
- **Response-family map**: the current user-facing response surface closes four practical serializer families: F (TWAP), G (error branch for hybrid actions), H (status-only success), and J (order-status fill family).
- **Fill hash payload**: fills hash through a separate 18-field payload: `px`, `sz`, `startPosition`, `dir`, `closedPnl`, `oid`, `crossed`, `fee`, `builderFee`, `tid`, `cloid`, `liquidation`, `feeTrialEscrow`, `builder`, `twapId`, `deployerFee`, `liquidatedUser`, `markPx`.
- **API wrapper fields**: `coin` and `feeToken` belong to API/user-fill wrapper surfaces, not to the hashed fill payload itself.

Current open lane:

- exact local derivation of fill economics and flags (`startPosition`, `closedPnl`, `tid`, liquidation membership, maker/taker routing)
- remaining mainnet-specific serializer families that are still not fully wired locally

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

## 8. Bridge Finalization and Validator-Set Updates

`CONFIRMED` from Bridge2 state naming, bridge action inventory, and the current
control-plane notes.

- **Bridge2 state** includes `withdrawal_signatures`,
  `withdrawal_finalized_votes`, `finished_withdrawal_to_time`,
  `validator_set_signatures`, `validator_set_finalized_votes`, `bal`,
  `last_pruned_deposit_block_number`, and `oaw` (`bool`).
- **Withdrawal entrypoint**: `withdraw3`.
- **Bridge signing action**: `ValidatorSignWithdrawal`.
- **Finalize actions**: `VoteEthFinalizedWithdrawal` and
  `VoteEthFinalizedValidatorSetUpdate`.
- **Validator-set update signing**: `SignValidatorSetUpdate`.
- **Signer model**: current repo truth keeps bridge signing on validator signer
  keys, not an unrelated bridge-only signer family.

What remains open is exact dispute-period closure and some Ethereum-side
operational details, not whether bridge finalization is a first-class staged
state machine.

---

## 9. Aligned Quote Token (HPT)

`DISCOVERED` from Hyperliquid docs + API.

- **Exchange field [56]** `hpt` = AlignedQuoteTokenTracker.
- **Per-token state**: isAligned, firstAlignedTime, evmMintedSupply, dailyAmountOwed, predictedRate.
- **Validator sidecar**: `aqa-publisher` ([github.com/native-markets/aqa-publisher](https://github.com/native-markets/aqa-publisher)).
- **Rate source**: 30-day SOFR average from NY Fed, FRED, OFR (median of 3 sources).
- **Submission**: `validatorL1Stream` action with `riskFreeRate` field, daily at 22:00 UTC.
- **Protocol rate**: Stake-weighted median of all active validator votes.
- **Fee benefits**: 20% lower taker, 50% better maker rebate for aligned stablecoins.

This is also a begin-block surface: `update_aligned_quote_token` is the current effect-9 lane in the widened testnet note.

---

## 10. Outcomes and Settlement Surface

`CONFIRMED` for the baseline market model; `HYPOTHESIS` for the leading
solvency-risk lane.

- **Risk mode**: 1x isolated-only, no leverage, no funding.
- **Settlement authority**: `oracleUpdater` posts final outcome via
  `SettleOutcome`.
- **Post-settlement behavior**: open orders auto-cancel via
  `outcomeSettledCanceled`; new orders reject via `outcomeSettledRejected`.
- **Question metadata**: `fallbackOutcome`, `namedOutcomes`,
  `settledNamedOutcomes`.
- **User token operations**: `SplitOutcome`, `MergeOutcome`, `MergeQuestion`,
  `NegateOutcome`.

Current open risk lane:

- `MergeQuestion` / fallback reconciliation is still the leading higher-order
  solvency hypothesis.
- The repo should treat question-level accounting as riskier than the simpler
  YES/NO pair mechanics until the binary path is closed.

---

## 11. Gossip Protocol

`CONFIRMED` from binary RE.

- **Frame format**: `[u32 BE len][u8 kind][body]`. Traced data frames often use LZ4 with `kind=0x01`, while greetings use `kind=0x00`.
- **Port 4001 greeting**: bincode-fork `TcpGreeting` with chain-based variant index. Current live observations are testnet variant `3` (8 bytes total) and mainnet variant `2` (7 bytes total).
- **Broader chain enum**: the ABCI-stream chain mapping still tracks `Local=0`, `Sandbox=1`, `Testnet=2`, `Mainnet=3`; do not conflate that with the observed 4001 `TcpGreeting` variant index.
- **Port 4003 authenticated greeting**: `keccak256("Hyperliquid Consensus Payload" + 0x00 + bincode(content))`; content is bincode, not msgpack.
- **Port 4003 signed content**: the greeting is variable-sized, not a tiny fixed blob. Captured content closes a 20-byte validator address, varint round, 1-byte variant flag, optional 32-byte ABCI extra, several additional counters plus a 32-byte hash, a sorted `heartbeat_statuses` map, and a trailing empty-mempool varint. A few middle counters still lack semantic labels.
- **Connection**: Server-side `connection_checks` — unknown peers rejected after 5s timeout.
- **Loopback**: Dominated by `MsgConcise`/`OutMsgConcise` (tag 27/28).

---

## 12. Guard System

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

## 13. Implementation Status

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
