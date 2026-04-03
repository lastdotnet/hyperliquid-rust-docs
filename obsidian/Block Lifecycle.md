# Block Lifecycle — Complete Hook Map

Every hook point in the Hyperliquid L1 block processing pipeline, from consensus proposal to state commitment. This is the authoritative reference for the hlx reimplementation.

**Source**: Binary RE of hl-node (VA 0x01e748e0), confirmed against testnet replay.

## Overview

```
HyperBFT Propose (leader)
    │
    ▼
RecoverUsers        ── signature recovery, broadcaster authorization
    │
    ▼
begin_block()       ── 9 named effects + supplementary per-book maintenance
    │
    ▼
DeliverSignedActions ── per-action dispatch through matching engine
    │
    ▼
ProcessFills        ── fee computation, clearinghouse settlement, state hash
    │
    ▼
EVM Block Phase     ── small block (1s/2M gas) + big block (60s/30M gas)
    │
    ▼
end_block()         ── mark price updates from book mids
    │
    ▼
Commit              ── state hash finalization, RespHash accumulator
    │
    ▼
VoteAppHash         ── validators compare state hashes (every 2000 blocks)
```

## Phase 1: RecoverUsers

Before any effects run, the proposer's block is validated:

| Step | What happens | Binary evidence |
|------|-------------|-----------------|
| Parse block header | Extract round, time, hardfork version, proposer | `process_block` line 1647 |
| Validate proposer | Check proposer is active validator | `HlBlockValidator::validate_header` |
| Validate broadcasters | Each bundle's broadcaster must be whitelisted | `BroadcasterSet::is_authorized` |
| Recover signers | EIP-712 typed data signature recovery per action | `resolve_actor` → k256 recovery |
| Nonce check | Reject duplicate nonces per address | `agents.use_nonce(user, nonce)` |

## Phase 2: begin_block() — 9 Named Effects

**CONFIRMED order from binary VA 0x01e748e0 (2026-04-02)**:

### Effect 1: UpdateOracle
- **What**: Pre-process oracle state for the new block
- **Guard**: None (runs every block)
- **State touched**: `oracle.pxs`, `clearinghouse.oracle_prices`
- **Note**: Actual price updates come from SetGlobal actions in the block; this slot handles any per-block oracle pre-processing

### Effect 2: DistributeFunding
- **What**: Settle accumulated funding payments to all open positions
- **Guard**: `funding_distribute_guard` (interval: 8000ms)
- **Formula**: `payment = position.szi * oracle_px * (cum_funding_global - pos.cum_funding_entry)`
- **State touched**: `clearinghouse.balances` (debited/credited), `cum_funding_per_asset`, each position's `cum_funding_entry`
- **Direction**: Positive rate = longs pay shorts

### Effect 3: ApplyBoleLiquidations
- **What**: Check BOLE lending positions for health factor < 0.8, execute liquidations
- **Guard**: None (runs every block if candidates exist)
- **Threshold**: Health factor < 0.8 (confirmed from binary constant)
- **Cooldown**: 30 seconds for partial liquidations
- **Types**: Partial (50% seize), Market (full repay), Backstop (default liquidator)
- **State touched**: `bole.reserves`, `bole.user_positions`, `bole.last_partial_liquidation`
- **ADL cascade**: Triggers `execute_adl()` if shortfall persists after BOLE liquidations

### Effect 4: UpdateFundingRates
- **What**: Sample premium and update per-asset funding rate EMA
- **Guard**: `funding_update_guard` (interval: 8000ms)
- **Formula**: DeterministicEma with fraction-based accumulation (num/denom)
- **State touched**: `funding[asset].premium_ema`, `funding[asset].velocity_ema`
- **Binary assertions**: `assert!(decay <= 1.)`, `assert!(self.denom > 0.)`

### Effect 5: RefreshHip3StaleMarkPxs
- **What**: Refresh mark prices that haven't been updated recently (fall back to oracle)
- **Guard**: `hip3_stale_mark_guard.guard` (interval from snapshot)
- **Threshold**: `staleness_threshold_ms` (from snapshot, typically 10000ms)
- **State touched**: `clearinghouse.mark_prices`, `hip3_stale_mark_guard.last_refresh_time`

### Effect 6: PruneBookEmptyUserStates
- **What**: Remove users with zero-size positions (fully closed)
- **Guard**: `book_empty_user_states_pruning_guard` (interval: 60000ms)
- **State touched**: `clearinghouse.positions` (entries removed)

### Effect 7: UpdateStakingRewards
- **What**: Stage new staking rewards for current epoch
- **Guard**: None
- **State touched**: `staking.rewards` (accumulated per-validator)

### Effect 8: UpdateActionDelayer
- **What**: Drain matured CoreWriter delayed actions and execute them
- **Guard**: None (runs every block)
- **Flow**: `action_delayer.drain_matured(block_time_ms)` → `dispatch_typed_action()` per matured entry
- **State touched**: `action_delayer.delayed_actions`, `action_delayer.n_total_delayed_actions`, plus whatever the executed actions touch
- **Note**: This is where EVM→L1 CoreWriter actions actually fire. They were enqueued in the PREVIOUS block's EVM phase and mature after `delay_scale * 3000ms`.

### Effect 9: UpdateAlignedQuoteToken
- **What**: Sample the stake-weighted median of validator risk-free rate votes
- **Guard**: None
- **State touched**: `hpt.current_risk_free_rate`
- **Data source**: Validators submit SOFR-based rates via `validatorL1Stream` action using the `aqa-publisher` sidecar. Rates are published daily at 22:00 UTC.

### Supplementary Per-Book Maintenance

After the 9 named effects, supplementary maintenance runs:

| Effect | Guard | What |
|--------|-------|------|
| **CheckTriggerOrders** | None | Fire TP/SL orders when oracle crosses trigger price |
| **CancelAggressiveOrdersAtOiCap** | `cancel_oi_cap_guard` (1000ms) | Cancel orders that would breach per-asset OI caps |
| **ValidatorL1VoteTrackerPruneExpired** | Internal to tracker (60000ms) | Remove expired voting rounds |
| **ResetRecentOis** | `reset_recent_oi_guard` (1000ms) | Reset HIP-3 recent OI windows |
| **UpdateStaleMarkGuards** | Per-book | Update per-book staleness timestamps |

## Phase 3: DeliverSignedActions

Each action bundle is processed sequentially:

```
for bundle in signed_action_bundles:
    for signed_action in bundle.signed_actions:
        1. Resolve actor (EIP-712 recovery or broadcaster fallback)
        2. Nonce replay protection
        3. Try typed dispatch (serde → HlAction enum)
        4. Fall back to JSON string dispatch
        5. Accumulate fills, cancels, blocked actions
```

### Action Categories (90 types)

| Category | Types | Key actions |
|----------|-------|-------------|
| **Trading** | 7 | Order, Cancel, CancelByCloid, Modify, BatchModify, ScheduleCancel, TwapOrder |
| **Transfers** | 6 | UsdSend, SpotSend, UsdClassTransfer, SendAsset, SubAccountTransfer, SubAccountSpotTransfer |
| **Margin** | 3 | UpdateLeverage, UpdateIsolatedMargin, TopUpIsolatedOnlyMargin |
| **Vault** | 5 | CreateVault, VaultModify, VaultTransfer, VaultDistribute, NetChildVaultPositions |
| **Staking** | 5 | TokenDelegate, LinkStakingUser, ClaimRewards, RegisterValidator, ExtendLongTermStaking |
| **Validator** | 9 | CValidator, CSigner, ValidatorL1Vote, ValidatorL1Stream, SignValidatorSetUpdate, VoteEthDeposit, etc. |
| **Account** | 7 | ApproveAgent, SetDisplayName, SetReferrer, ConvertToMultiSigUser, ApproveBuilderFee, StartFeeTrial, CreateSubAccount |
| **DeFi** | 4 | BorrowLend, UserDexAbstraction, UserPortfolioMargin, SendToEvmWithData |
| **Governance** | 3 | GovPropose, GovVote, VoteGlobal (22 sub-types) |
| **HIP-3/4** | 3 | PerpDeploy (8 sub-types), SpotDeploy, SetGlobal (6 sub-types) |
| **System** | 11 | SystemBole, SystemSpotSend, CWithdraw, CUserModify, EvmUserModify, EvmRawTx, etc. |
| **Special** | 6 | Noop, Withdraw3, MultiSig, Liquidate, UserOutcome (4 sub-types), ForceIncreaseEpoch |

### Dispatch Flow

```
HlAction::Order(a) ─────────► matching.process_order()
                                  │
                                  ▼
                            OrderResult { fills, resting }
                                  │
                          ┌───────┴───────┐
                          ▼               ▼
                    Fill recorded    Order rests in book
                    (to result.fills)  (oid_to_order index)
                          │
                          ▼
              state_hasher.record_fill_response()

HlAction::Cancel(a) ────────► matching.cancel_order_by_oid()
                                  │
                                  ▼
                    state_hasher.record_cancel_response()

HlAction::Liquidate(a) ─────► build_user_margin_state()
                                  │
                                  ▼
                          is_liquidatable()?
                          ┌──────┴──────┐
                          ▼             ▼
                     No: reject    Yes: transfer position
                                       credit PnL
                                       record_liquidation()
```

## Phase 4: ProcessFills

After all actions in the block, accumulated fills are processed:

| Step | What happens |
|------|-------------|
| **Fee computation** | Per-fill: `compute_fee()` for maker and taker using fee schedule + tiers + referral |
| **Fee debit** | Debit fees from user balances |
| **Volume tracking** | Record volume in `fee_tracker` for tier progression |
| **Clearinghouse settlement** | `clearinghouse.process_fill()` — update positions, PnL |
| **Fill counter** | Increment `total_fills`, per-asset stats |
| **Fill history** | Store in `user_fills` (capped at 1000 per user) |
| **ExEx events** | Emit `ExchangeEvent::Fill` for extensions |
| **State hash** | Fill responses recorded in LtHash16 accumulators |

## Phase 5: EVM Block Phase

```
build_small_evm_block_and_apply_l1_effects()  ── 1s / 2M gas
    │
    ▼
Scan receipts for CoreWriter RawAction logs
    │
    ▼
build_big_evm_block_and_apply_l1_effects()    ── 60s / 30M gas
    │
    ▼
Scan receipts for CoreWriter RawAction logs
    │
    ▼
enqueue_core_writer_action() for each found   ── goes to action_delayer
```

**CoreWriter flow**: EVM contracts call `sendRawAction(bytes)` on the precompile at `0x3333...3333`. The calldata encodes one of 15 action types (limit order, vault transfer, token delegate, etc.). Decoded actions are enqueued in the `action_delayer` with a configurable delay (`delay_scale * 3000ms`). They fire in Effect 8 of the NEXT block's begin_block.

**In replay mode** (no EVM executor): raw tx calldata is scanned directly for CoreWriter calls via RLP parsing. Actions are decoded and enqueued without EVM execution.

**L1→EVM transfers**: Pending transfers in `hyper_evm.l1_to_evm_native_token_queue` and `l1_to_evm_erc20_queue` are processed here.

## Phase 6: end_block()

| Step | What happens |
|------|-------------|
| **Mark price update** | For each order book with orders, set `mark_prices[asset] = book.mid_price()` |

## Phase 7: Commit + VoteAppHash

Every 2000 blocks, validators submit `VoteAppHash` actions containing their computed state hash. The protocol tracks votes in `app_hash_vote_tracker` and declares quorum when 2/3+ of active validators agree.

### State Hash Algorithm

```
For each of 14 L1 categories + 3 EVM categories:
    1. Serialize the state element via rmp_serde (named-key msgpack)
    2. Hash with blake3 XOF (2048 bytes output)
    3. Accumulate into LtHash16 (paddw u16 lattice hash)
    4. SHA-256 the final 2048-byte LtHash16 digest → 32 bytes
```

The 14 L1 categories include: order responses, cancel responses, fill responses, liquidation responses, funding settlements, oracle updates, staking state, vault state, clearinghouse state, etc.

## Timing

| Parameter | Value | Source |
|-----------|-------|--------|
| Block time | ~70ms (HyperBFT) | Binary default |
| Funding interval | 8000ms | `funding_distribute_guard` |
| OI cap check interval | 1000ms | `cancel_oi_cap_guard` |
| Book pruning interval | 60000ms | `book_empty_user_states_pruning_guard` |
| Vote tracker prune interval | 60000ms | `validator_l1_vote_tracker` |
| HIP-3 stale mark check | From snapshot | `hip3_stale_mark_guard` |
| Recent OI reset interval | 1000ms | `reset_recent_oi_guard` |
| CoreWriter delay | 3000ms * delay_scale | `action_delayer` |
| BOLE partial liquidation cooldown | 30s | `partial_liquidation_cooldown` |
| VoteAppHash checkpoint interval | 2000 blocks | Binary constant |
| Aligned quote token vote | Daily at 22:00 UTC | `aqa-publisher` sidecar |

## Guard System

All rate-limited effects use `BucketGuard`:
```rust
pub struct BucketGuard {
    pub last_time: Option<u64>,   // Last trigger time (ms)
    pub interval_ms: u64,         // Minimum interval between triggers
}
```

`check(now_ms)` returns true if `now_ms - last_time >= interval_ms`, then advances `last_time`. Guards are loaded from snapshots during bootstrap, preserving the exact timing state from the real node.
