# HyperBFT Protocol Specification

A reverse-engineered specification of the Hyperliquid L1 consensus and execution protocol. Derived from binary analysis of the deployed `hl-node` validator binary, live traffic capture, and on-chain observation.

**Status**: Living document. All claims labeled CONFIRMED (from disassembly/binary strings), OBSERVED (from live traffic), or INFERRED (from pattern matching). Updated 2026-04-02.

---

## 1. Overview

Hyperliquid L1 (HyperCore) is a high-performance blockchain optimized for financial applications. It runs a modified HotStuff BFT consensus protocol ("HyperBFT") with an integrated matching engine, lending system (BOLE), and EVM execution layer (HyperEVM).

### 1.1 Design Goals
- Sub-second block times (~1 block/sec observed on testnet)
- Deterministic execution (no MEV within a block)
- Integrated order matching at the consensus layer
- Dual execution: L1 matching engine + EVM smart contracts
- Fully collateralized outcome contracts (HIP-4)

### 1.2 Key Parameters (Testnet)
- Active validator set: top 50 by stake
- Epoch-based validator rotation
- Two-chain commit rule (2 consecutive QCs for finality)
- ~200,000 orders/sec platform-wide capacity (claimed)

---

## 2. Consensus: HyperBFT

### 2.1 Protocol Family

HyperBFT is a **two-chain HotStuff** variant (CONFIRMED from binary string `"CertifiedTwoChainCommit"`). It is NOT three-phase classical PBFT.

**Consensus message signing**: secp256k1 (k256-0.13.3 / secp256k1-0.29.0), NOT ed25519. Domain separator: `"Hyperliquid Consensus Payload"`.

### 2.2 Round Lifecycle

Each round follows this cycle:

```
1. PROPOSE     Proposer assembles block from mempool
2. VOTE        Validators verify and sign vote
3. QC          2/3+ stake-weighted votes → Quorum Certificate
4. COMMIT      Two consecutive QCs → block finalized (two-chain rule)
5. ADVANCE     Round increments, next proposer selected
```

If the proposer fails to propose within the timeout:

```
1. TIMEOUT     Validator broadcasts timeout message
2. TC          2/3+ timeouts → Timeout Certificate
3. ADVANCE     Round advances without a committed block
```

### 2.3 Proposer Selection

Modified round-robin with stake weighting (CONFIRMED: `RoundRobinTtl`).

- Proposer is deterministic from round number and validator stakes
- `next_proposers` (plural) — the system precomputes upcoming proposers
- Jailed validators are excluded from proposer rotation
- If proposer is suspected (missed heartbeat/timeout), a TC triggers rotation via the `suspect` mechanism

### 2.4 Block Structure

**Internal Block** (consensus layer):
```
Block {
    round:        u64       — monotonically increasing
    parent_round: u64       — previous block's round
    tx_hashes:    Vec<Hash> — ordered list of transaction hashes
    qc:           Qc        — Quorum Certificate for parent block
    tc:           Option<Tc> — Timeout Certificate (if previous round timed out)
    block_hash:   Hash      — hash of this block's contents
    time:         Timestamp — block timestamp
    proposer:     Address   — proposer validator address
}
```

**ClientBlock** (broadcast to non-validator peers):
```
ClientBlock {
    proof: {
        signed_block:  Signed<Block>   — the committed block with proposer's signature
        commit_proof:  CommitProof      — child block + grandchild QC
    }
    txs: Vec<SignedAction>              — full transaction bodies (resolved from tx_hashes)
}
```

Key difference: `Block` contains `tx_hashes` (just hashes). `ClientBlock` contains full `txs` (the actual signed action bodies resolved from mempool).

### 2.5 Quorum Certificate (QC)

```
Qc {
    block_hash:  Hash
    round:       u64
    signatures:  Vec<(Validator, Signature)>
}
```

Quorum threshold: 2/3+ of total stake weight (standard BFT).

Validation errors:
- `QcNoQuorum` — insufficient stake weight
- `QcRoundBeforeHardfork` — QC from before hardfork boundary
- `EmptyValidators` — no validators to check against

### 2.6 Two-Chain Commit Rule

A block B at round R is **committed** when:
1. Block B has a QC (is **Certified**)
2. Child block C at round R+1 also has a QC
3. C's QC references B (C.parent_round = R)

The **CommitProof** contains:
```
CommitProof {
    child:          Signed<Block>  — the child block
    grandchild_qc:  Qc             — QC of the grandchild (proves child was certified)
}
```

Validation:
- `CommitProofChildQc` — child's QC must reference the committed block
- `CommitProofGrandchildQc` — grandchild's QC must reference the child
- `CommitProofConsecutive` — blocks must be from consecutive rounds

### 2.7 Timeout Certificate (TC)

When a proposer fails, validators broadcast timeouts:

```
Timeout {
    round:  u64
    reason: enum  — why the timeout occurred
}

Tc {
    signed_timeouts: Vec<(Validator, SignedTimeout)>
    round:           u64
}
```

The TC is included in the next block's `tc` field, proving the timeout was legitimate.

Validation:
- `TcNoQuorum` — insufficient stake weight
- `TcNoTimeout` — TC doesn't contain a timeout
- `TimeoutRoundMismatch` — round doesn't match
- `AlreadyHaveTimeout` — duplicate from same validator

### 2.8 Validator Signing Gates

Before signing a vote, a validator checks:
1. **`is_main_signer`** — is this the designated signer for this validator?
2. **Active set** — validator must be in the top N by stake
3. **Not jailed** — checked against `jailed_signers`
4. **`last_vote_round`** — prevents double-voting in the same round
5. **Not disabled** — checked against `disabled_validators`
6. **Round match** — vote round must match current consensus round

If the validator leaves the active set: `"Home validator has left the active set. May not have enough stake to be in active set."`

---

## 3. Mempool

### 3.1 Structure

```
Mempool {
    block_hash_to_block:    HashMap<Hash, SignedBlock>
    committed_tx_hashes:    HashSet<Hash>       — dedup against committed txs
    uncommitted_txs:        Vec<Tx>             — pending transactions
    tx_hash_to_seq_num:     HashMap<Hash, u64>  — ordering
    blocks:                 Vec<Block>          — block buffer
    rpc_requests:           Queue               — pending RPC requests
}
```

### 3.2 Network-Level Filtering

Before reaching the mempool, connections pass through:
1. **Max peers** — hard cap on simultaneous TCP connections
2. **Validator/sentry check** — `firewall_ips` cross-checked against active validator set
3. **Rate limiting** — `rate_limiter_bytes_per_sec` with soft/hard thresholds
4. **ABCI state request rate limit** — per-IP throttle on bootstrap requests

Rate limiter uses `BucketGuard` primitive (`base/src/bucket_guard.rs`): `{last_bucket, bucket_millis}`.

### 3.3 Transaction Ingress

Two ingress paths converge at `add_tx`:
- **External TX** (from clients/RPC): `external_tx_recver` → `forward_external_txs`
- **Gossip** (from validators): `MsgConcise::Tx` (variant 0)

### 3.4 Transaction Admission

Transactions are bundled by ~8 whitelisted **broadcasters**. Each bundle contains:
```
SignedActionBundle {
    signed_actions:     Vec<SignedAction>    — user-signed actions
    broadcaster:        Address             — broadcaster's address
    broadcaster_nonce:  u64                 — monotonically increasing per broadcaster
}
```

**Admission checks** (7 stages, in order):
1. Parse action — `"could not parse action"` / `"unsupported chain"` → reject
2. Signature verify — `AddTxNotSigned` → reject
3. Broadcaster whitelist — `TxUnexpectedBroadcaster` → reject
4. Broadcaster nonce — `TxInvalidBroadcaster { Nonce, err }` → reject (must be sequential)
5. Dedup by hash — `AddTxDuplicate { tx_hash, existing_from_rpc }` → reject
6. Already committed — `AddTxCommitted` → reject
7. Enqueue → `uncommitted_txs` + `tx_hash_to_seq_num`

**No hard mempool size limit** — eviction is time-based via periodic `mempool refresher`, not capacity-based.
**No per-user throttle in mempool** — rate limiting at network layer (`rate_limiter`) and exchange layer (`ensure_order_bucket_guard`).

### 3.5 Transaction Ordering

Transactions are ordered by:
1. `tx_hash_to_seq_num` — sequence number assignment at admission
2. Builder priority — `GossipPriorityBidAction` / `GossipPriorityGasAuction`
3. Broadcaster bundling order

### 3.6 Eviction

Periodic `mempool refresher` (timer-driven, not event-driven):
- `dropping txs` — evicts stale uncommitted transactions
- `dropping blocks` — evicts stale blocks
- `Pruned rpc request` — removes old RPC entries
- Size stats logged periodically

---

## 4. Block Execution

### 4.1 ABCI Pipeline

```
ClientBlock received
    → verify CommitProof
    → run_node_apply_abci_block()
    → begin_block_logic_guard
    → for each signed_action_bundle:
        for each signed_action:
            → deliver action (process through Exchange state machine)
    → end_block (l1/src/exchange/end_block.rs)
    → build EVM blocks
    → compute app_hash
    → broadcast VoteAppHash
```

### 4.2 Exchange State Machine

The Exchange struct has **57 fields** and processes **80 action variants**.

Key subsystems:
- Order matching (BTreeMap + VecDeque books, O(1) cancel)
- Funding settlement (EMA premium, rate expressed per 8h, settled every 1h at rate/8 — CONFIRMED settlement interval from live API, INFERRED rate division)
- Staking / delegation (CStaking with 23 fields)
- Lending (BOLE — BolePool with 19 fields)
- Outcome contracts (HIP-4)
- Vault management
- Sub-accounts
- Agent authorization

### 4.3 EVM Block Building

Two EVM block types per L1 block:
- **Small block** (`build_small_evm_block_and_apply_l1_effects`) — every L1 block
- **Big block** (`build_big_evm_block_and_apply_l1_effects`) — periodic, larger gas limit

L1→EVM effects:
- Native HYPE token transfers via `l1_to_evm_native_token_queue`
- ERC-20 transfers via `l1_to_evm_erc20_queue`
- System transactions via CoreWriter precompile (address `0x3333...3333`)

EVM stack: revm 36, Cancun spec.

### 4.4 State Hashing

**Algorithm** (CONFIRMED):
```
rmp_serde → blake3 XOF (2048B) → paddw u16 → SHA-256 (32B)
```

**LtHash16**: Lattice-based homomorphic hash. 1024 × u16 values = 2048 bytes per accumulator. Supports incremental add/remove without full recomputation.

**14 total accumulators (11 L1 + 3 EVM)**:
```
CValidator, CSigner, Na, LiquidatedCross, LiquidatedIsolated, Settlement,
NetChildVaultPositions, SpotDustConversion, BoleMarketLiquidation,
BoleBackstopLiquidation, BolePartialLiquidation
```

**EVM hash format**: Raw byte concat (92B accounts, 84B storage, variable contracts).

**VoteAppHash**: Every 2000 blocks, validators vote on state hash. Quorum tracked in `app_hash_vote_tracker`. Mismatch saves state to `/tmp/abci_state_with_quorum_app_hash_mismatch.rmp`.

---

## 5. Networking: Gossip Protocol

### 5.1 Ports

| Port | Purpose |
|------|---------|
| 4001 | Block streaming (abci_stream) |
| 4002 | Peer discovery / RPC verification |
| 4003 | External transaction submission |
| 4004 | Additional validator communication |

### 5.2 Wire Format

```
[u32 BE body_len][u8 kind][body]
```

- `kind=0x01`: LZ4-compressed bincode body
- `kind=0x00`: raw/uncompressed

After LZ4 decompression, the body is bincode-fork encoded with varint integers.

### 5.3 Connection Handshake

**CORRECTED 2026-04-02**: The greeting is a **bincode-fork serialized `TcpGreeting` enum** with 2 variants (`send_abci`, `broadcast_group`), framed with u32 BE length prefix. Chain enum: Local=0, Sandbox=1, Testnet=2, Mainnet=3.

Wire format: `[u32 BE len][u32 LE variant_index][u8 bool send_abci][varint u64 id]`

Server-side `connection_checks` verify the connecting IP is a known validator/sentry. Times out after 5 seconds for unknown peers.

After handshake:
1. Peer verification via port 4002 back-connect (`connection_checks`)
2. ABCI state transfer (~1.16GB uncompressed)
3. EVM KV transfer (~4GB RocksDB snapshot via EvmKvsMsg)
4. Block streaming begins

### 5.4 Consensus Message Types

**Tag 27 = MsgConcise** (inbound, 8 variants):

| Index | Variant | Purpose |
|-------|---------|---------|
| 0 | Tx | Transaction |
| 1 | Block | Block proposal |
| 2 | Vote | Vote on block |
| 3 | Timeout | Timeout vote |
| 4 | Tc | Timeout Certificate |
| 5 | BlocksAndTxs | Batch sync |
| 6 | Heartbeat | Validator liveness |
| 7 | HeartbeatAck | Heartbeat response |

**Tag 28 = OutMsgConcise** (outbound, 9+ variants):

| Index | Variant | Purpose |
|-------|---------|---------|
| 0 | Block | Block forward |
| 1 | ClientBlock | Committed block for clients |
| 2 | VoteJailBundle | Jail vote |
| 3 | Timeout | Timeout forward |
| 4 | Tc | TC forward |
| 5 | Heartbeat | Heartbeat forward |
| 6 | NextProposer | Next proposer + QC/TC |
| 7 | Tx | Transaction forward |
| 8 | RpcRequest | RPC request |

**MsgConcise outer struct**: `{source, msg (mid-tagged), prev_round, reason}`

**Mid field** (consensus flow direction):
```
0 = in              — inbound consensus message
1 = execution state  — state after execution
2 = out             — outbound consensus message
3 = status          — consensus status snapshot
4 = round advance   — round transition
```

### 5.5 Heartbeat

Heartbeats carry validator liveness information and state snapshots:

```
Heartbeat payload:
    validator:                    Address (20 bytes)
    round:                        u64
    HeartbeatSnapshot:
        ValidatorSetSnapshot:
            stakes:               Vec<(Validator, Stake)>
            jailed_validators:    Vec<Validator>
        evm_block_number:         u64
        ConciseLtHashes:
            accounts_hash:        ConciseLtHash {hash_concise: 32B SHA-256, n_elements, n_bytes}
            contracts_hash:       ConciseLtHash
            storage_hash:         ConciseLtHash
    validator_to_last_msg_round:  HashMap<Validator, Round>
    random_id:                    u64
```

**Critical**: `hash_concise` is 32 bytes (SHA-256 digest of full LtHash16), NOT the raw 2048-byte accumulator. Full accumulators only available from ABCI state snapshots.

### 5.6 Jailing via Heartbeat

Validators monitor peer liveness via heartbeats:
- `validator_latency_ema` — exponential moving average of response time
- `latency_ema_jail_threshold` — if EMA exceeds threshold, propose jailing
- `validators_missing_heartbeat` — set of non-responsive validators
- `disconnected_validators` — detected network partitions

Jailing flow:
1. `JailVoteTimer` fires periodically
2. Check latency EMA against threshold
3. Construct `VoteJailBundle` with candidates
4. If 2/3+ validators agree → validator jailed
5. Jailed validator can unjail via `CSignerAction::unjailSelf` after cooldown

### 5.7 RPC Backfill (Peer Sync)

When a node is behind, it uses `BlocksAndTxs` RPC to catch up.

**Lag detection**: Node queries ≥2 peers for `GossipStatus { initial_height, latest_height }`. Local state classified as:
- **fresh** — small gap, use incremental `BlocksAndTxs` RPC
- **stale** — too far behind or wrong hardfork, need full bootstrap
- **missing** — no local state, full bootstrap

**RpcRequestContent**:
```
BlockTx                                         — single block request
BlocksAndTxs { after_round, n, last_block_hash } — range backfill
```

**Peer selection**: Round-robin over active validator set (`rpc_clone_main_validators` → `rpc_task_get_peers`). Try each in order; if all fail: `RpcRoundRobin` error.

**Full bootstrap** (stale/missing path):
1. Connect to peer port 4001, exchange TCP greeting
2. Peer sends full ABCI state (~500MB) + EVM KVs (~4GB RocksDB)
3. State validated against quorum app hash
4. Block backfill from state height to tip via BlocksAndTxs

**Response validation** (9 checks on every backfilled ClientBlock):

| # | Check | Error |
|---|-------|-------|
| 1 | QC round matches block round | `ClientBlockQcRound` |
| 2 | QC hash matches computed block hash | `ClientBlockQcHash` |
| 3 | Each transaction is valid | `ClientBlockTx` |
| 4 | tx_hashes == hash(txs) | `ClientBlockTxHashes` |
| 5 | Timestamp monotonically increasing | `ClientBlockTime` |
| 6 | Commit proof present | `ClientBlockMissingCommitProof` |
| 7 | Child QC references parent | `CommitProofChildQc` |
| 8 | Grandchild QC references child | `CommitProofGrandchildQc` |
| 9 | Rounds are consecutive (R, R+1, R+2) | `CommitProofConsecutive` |

**Transition to live**: After backfill → `forward_client_blocks` streaming loop on port 4001. Periodic `mempool refresher` fires gap-fill RPC requests.

Safeguards: `"too many blocks to request"`, `"bootstrap ended past original block, is peer malicious?"`.

---

## 6. Validator Operations

### 6.1 Staking

- **CStaking** (23 fields): on-chain staking contract
- **Staking** (6 fields): consensus-layer epoch management
- Active set: top N validators by total delegation
- Epoch transitions via `ForceIncreaseEpoch` action
- `self_delegation_requirement` enforced
- Long-term staking with lock periods

### 6.2 Key Model

Two keys per validator:
- **Validator key** (cold): holds funds, receives rewards, manages profile
- **Signer key** (hot): signs consensus messages, stored in `node_config.json`

### 6.3 Privileged Actions

**CSignerAction** (3 variants, requires hot signer key):
- `unjailSelf` — self-unjail after cooldown
- `jailSelf` — voluntary jail before shutdown
- `jailSelfIsolated` — jail in isolated mode

**CValidatorAction** (16 variants, requires cold validator key):
- Asset registration (registerAsset, registerAsset2)
- Oracle management (setOracle)
- Fee configuration (setFeeRecipient, setFeeScale)
- Margin tables (insertMarginTable, setMarginTableIds)
- Open interest caps
- Funding rates
- DEX management (disableDex, setPerpAnnotation)

### 6.4 Bridge

Validators participate in the Ethereum bridge:
- `VoteEthDeposit` — vote to confirm L1 deposit
- `ValidatorSignWithdrawal` — sign withdrawal
- `SignValidatorSetUpdate` — sign validator set change
- `VoteEthFinalizedWithdrawal` — finalize withdrawal

Bridge state tracked in `Bridge2` struct (10 fields).

---

## 7. Application Layer

### 7.1 Action Types

The main Action enum has **90 variants** covering:
- Order placement/cancellation/modification
- Transfers (USD, spot, cross-chain)
- Staking/delegation
- Vault management
- Governance proposals
- HIP-3 deployer operations
- HIP-4 outcome operations
- BOLE lending
- Multi-sig
- EVM interactions

### 7.2 HIP-3: Builder-Deployed Perpetuals

Deployers stake HYPE and configure:
- `collateralToken` — any aligned quote token
- `oracleUpdater` — designated oracle address
- `feeRecipient` — custom fee destination
- `sub_deployers` — delegated deployment authority
- Rate limited: gas auction for deployment slots

### 7.3 HIP-4: Outcome Contracts

Binary instruments settling between 0 and 1:
- **1x isolated margin only** — no leverage, no liquidations
- **Collateral**: configurable per deployment (any aligned quote token)
- **Price discovery**: CLOB mid price only, no continuous oracle
- **Settlement**: deployer-designated `oracleUpdater` posts final outcome
- **Fractional settlement** via `settleFraction`

Token operations:
- `SplitOutcome` — lock collateral → mint YES+NO pair
- `MergeOutcome` — burn YES+NO → reclaim collateral
- `NegateOutcome` — flip YES↔NO atomically
- `MergeQuestion` — redeem complete set across multi-outcome question

Deployed through HIP-3 pipeline (`Hip3DeployAction` → `OutcomeDeploy`).

### 7.4 BOLE: Borrow/Lend

Protocol-controlled lending (NOT deployer-configurable):
- `BolePool` (19 fields) per token
- `SystemBole` — broadcaster-only pool operations
- Validator governance sets `Hip3BackstopLiquidatorParams`
- Three liquidation types: Market, Backstop, Partial

### 7.5 Governance

On-chain governance tied to CStaking:
- `GovPropose` (6 fields: title, description + 4)
- `GovVote` (5 fields)
- Delegation-weighted voting
- Proposal limits: max active proposals, title 1-50 chars, description 1-1000 chars

---

## 8. Durability Model

### 8.1 State Storage

- **L1 state**: in-memory only, periodic msgpack snapshots to disk (~500MB)
- **EVM state**: RocksDB (`EvmStateDbHome`)
- **Blocks**: NDJSON files (`replica_cmds/`)
- **EVM hashes**: JSON (`lt_hashes.json` per checkpoint)

### 8.2 Recovery

On crash/restart:
1. Load latest `periodic_abci_state` snapshot (.rmp)
2. Replay `replica_cmds` blocks from snapshot height to tip
3. Reconstruct full in-memory state
4. Resume consensus

---

## Appendix A: Key Function Addresses (Deployed Binary)

| Function | VA | Purpose |
|----------|-----|---------|
| MsgConcise serializer | `0x42b7e50` | Serialize inbound consensus msgs |
| OutMsgConcise serializer | `0x42b87e0` | Serialize outbound consensus msgs |
| ConciseMid serializer | `0x42b92f0` | Serialize mid-tagged enum |
| HeartbeatSnapshot serializer | `0x438fcc0` | Heartbeat payload |
| ConciseLtHash serializer | `0x43db480` | LtHash wire format |
| Core bincode-fork serializer | `0x474f3f0` | Writes tag as u32 LE |
| send_to_destination | `0x43bf670` | Route + serialize consensus msgs |
| BTreeMap peer routing | `0x409c0e0` | Peer dispatch by validator addr |
| tcp_bytes state machine | `0x1cc9d00` | TCP frame write (bswap for BE) |
| CSignerAction dispatch | `0x3352310` | Handle signer actions |
| Signer epoch check | `0x3355790` | "Signer invalid or inactive" |
| Validator lookup | `0x2eb8fb0` | Search epoch_states for signer |

## Appendix B: Source File Layout (from debug strings)

```
node/src/
├── node.rs                    — Node orchestrator
├── hl_node.rs                 — CLI entry point
├── consensus/
│   ├── state.rs               — Consensus state machine
│   ├── types.rs               — Qc, Tc, BlockHash, HeartbeatSnapshot
│   ├── mempool.rs             — Transaction mempool
│   ├── client_block.rs        — ClientBlock construction
│   ├── heartbeat_tracker.rs   — Liveness monitoring
│   ├── execution_state.rs     — Execution state bridge
│   ├── rpc.rs                 — Consensus RPC (RpcUpdate enum)
│   ├── server.rs              — Consensus server
│   ├── network.rs             — Peer connections
│   ├── signed.rs              — Signed<T> wrapper
│   ├── validator_set.rs       — Validator set management
│   ├── timer.rs               — Round timer
│   ├── config.rs              — Consensus config
│   ├── liner.rs               — Block ordering
│   ├── node_ips.rs            — Validator IP tracking
│   └── latency_samplers.rs    — Latency measurement
├── firewall_ips.rs            — Firewall IP management
└── startup.rs                 — Node startup

l1/src/
├── exchange/
│   ├── exchange.rs            — 56-field Exchange state
│   ├── end_block.rs           — End-block processing
│   └── impl_outcome.rs        — HIP-4 outcomes
├── abci/
│   ├── engine.rs              — ABCI engine
│   └── state.rs               — ABCI state
├── staking.rs                 — Staking module
├── bole/
│   └── user_state.rs          — BOLE user state
└── action/
    ├── vote_global.rs         — VoteGlobal handling
    └── rmp_signable.rs        — Action signing

evm_rpc/src/
└── evm_write_forwarder.rs     — EVM RPC forwarding

base/src/
├── production.rs              — Production mode
├── crit_msg.rs                — Crash handler
└── ren.rs                     — Visor (process supervisor)
```

---

## Appendix C: Evidence Map

Every structural claim in the hlx implementation traces to a confirmed binary finding.

### Consensus (bft.rs)

| Implementation Choice | Binary Evidence | Location |
|----------------------|-----------------|----------|
| Two-chain commit (not 3-phase) | `"CertifiedTwoChainCommit"` string | Binary rodata |
| `CommitProof = {child, grandchild_qc}` | `CommitProofChildQc`, `CommitProofGrandchildQc`, `CommitProofConsecutive` errors | VA 0x6600fa |
| Domain separator `"Hyperliquid Consensus Payload"` | Exact string | VA 0x76db94 |
| secp256k1 signing | `k256-0.13.3`, `secp256k1-0.29.0` crate versions | Binary dependency strings |
| `last_vote_round` double-vote prevention | `last_vote_round` field in `VoteJailBundle` | Binary serde strings |
| TC = `{signed_timeouts, round}` | `TcNoQuorum`, `TcNoTimeout`, `TimeoutRoundMismatch`, `AlreadyHaveTimeout` | VA 0x6600fa |
| Leader = stake-weighted round-robin | `RoundRobinTtl`, `next_proposers` (plural) | Binary strings |
| `BadBlockRound {block_round, last_commit_round}` | Exact error + field names | Binary strings |
| `BadBlockQcRound`, `BadBlockTcRound` | Exact error strings | Binary strings |
| `QcNoQuorum` = "Qc does not have quorum" | Exact string | Binary strings |
| `QcRoundBeforeHardfork` | Exact string | Binary strings |
| `BlockAlreadyRegistered` | Exact string | Binary strings |
| `DuplicateBlockRound {known_block, new_block}` | Exact error + fields | Binary strings |
| `TimeoutRoundMismatch` | Exact string | Binary strings |
| `AlreadyHaveTimeout` (per validator) | `AlreadyHaveTimeoutFromNode` string | Binary strings |
| `ValidatorIPMismatch {validator, node_ip, expected_ip}` | Exact error + fields | Binary strings |
| `ConsensusEvent` 6 variants | `ConsensusTxHash`, `ExecutionState`, `TriggerTimeout`, `EmptyBlockTimer`, `PeriodicVoteStream`, `JailVoteTimer` | Agent: consensus transition |
| Source: `node/src/consensus/state.rs` | Debug path embedded | Binary strings |

### Mempool (mempool.rs)

| Implementation Choice | Binary Evidence | Location |
|----------------------|-----------------|----------|
| `committed_tx_hashes: HashSet` | `committed_tx_hashes` field name | Binary serde strings |
| `uncommitted_txs` map | `uncommitted_txs` field name | Binary serde strings |
| `tx_hash_to_seq_num` ordering | `tx_hash_to_seq_num` field name | Binary serde strings |
| `block_hash_to_block` map | `assertion failed: self.block_hash_to_block.insert(block_hash, signed_block).is_none()` | Binary panic string |
| `AddTxDuplicate {tx_hash, existing_from_rpc}` | Exact error + fields | VA 0x6622cf |
| `AddTxCommitted` | Exact error string | Binary strings |
| `AddTxNotSignedAction` | Exact error string | Binary strings |
| `TxUnexpectedBroadcaster {broadcaster}` | Exact error + field | Binary strings |
| `TxInvalidBroadcasterNonce {Nonce, err}` | Exact error + fields | Binary strings |
| `broadcasters` whitelist in CStaking | `broadcasters` field, `VoteGlobal::ModifyBroadcaster` variant | Binary serde strings |
| No hard mempool size limit | No capacity/max_size string found in binary | Negative evidence |
| Time-based eviction | `mempool refresh`, `dropping txs`, `dropping blocks` | Binary strings |
| Source: `node/src/consensus/mempool.rs` | Debug path embedded | Binary strings |

### Block Building

| Implementation Choice | Binary Evidence | Location |
|----------------------|-----------------|----------|
| Block = `{round, parent_round, tx_hashes, qc, tc, block_hash, time, proposer}` | Field names in MsgConcise::Block serde | VA 0x660467 |
| ClientBlock has full `txs` (not just hashes) | `txs`, `proof_len`, `txs_len` in OutMsgConcise::ClientBlock | VA 0x6604cd |
| `build_small_evm_block_and_apply_l1_effects` | Exact function name string | Binary strings |
| `build_big_evm_block_and_apply_l1_effects` | Exact function name string | Binary strings |
| `l1_to_evm_native_token_queue` | Exact field name | Binary serde strings |
| `l1_to_evm_erc20_queue` | Exact field name | Binary serde strings |
| `begin_block_logic_guard` | Exact string | Binary strings |
| Source: `l1/src/exchange/end_block.rs` | Debug path embedded | Binary strings |

### Wire Protocol

| Implementation Choice | Binary Evidence | Location |
|----------------------|-----------------|----------|
| Tag 27 = MsgConcise (8 variants) | Jump table at VA 0x65f418, serializer at VA 0x42b7e50 | Agent: tag 27/28 |
| Tag 28 = OutMsgConcise (9+ variants) | Jump table at VA 0x65f458, serializer at VA 0x42b87e0 | Agent: tag 27/28 |
| Mid 0-4 = {in, execution state, out, status, round advance} | Jump table at VA 0x65f47c, 5 LEA branches to string labels | Agent: mid 0-4 |
| Frame format `[u32 BE len][u8 kind][LZ4 body]` | `bswap` at VA 0x1cc9f59, kind byte at VA 0x1cc9f5b | Agent: send path |
| `hash_concise` = 32 bytes SHA-256 (not 2048B LtHash16) | Hex loop at VA 0x449e0f0 processes exactly 32 bytes | Agent: heartbeat layout |
| `evm_block_number` in HeartbeatSnapshot | String at rodata 0x1de290, LEA at VA 0x42b9aa1 | Agent: heartbeat layout |
| ValidatorSetSnapshot = `{stakes, jailed_validators}` (2 fields, not 3) | Deserializer at VA 0x42b9b80, only 2 field dispatches | Agent: heartbeat layout |
| Source: `node::consensus::rpc::RpcUpdate` | `core::result::Result<node::consensus::rpc::RpcUpdate, node::consensus::error::Error>` | Binary type path string |

### Validator Operations

| Implementation Choice | Binary Evidence | Location |
|----------------------|-----------------|----------|
| CSignerAction = 3 variants (unjailSelf, jailSelf, jailSelfIsolated) | `variant index 0 <= i < 3` near CSignerAction | Binary serde strings |
| CValidatorAction = 16 variants | `variant index 0 <= i < 16` + full variant list | Binary serde strings |
| 90 total Action variants | `variant index 0 <= i < 90` | Binary serde strings |
| Epoch activation = top N by stake | Validator lookup at VA 0x2eb8fb0, `epoch_states[cur_epoch]` | Agent: signer epoch |
| `"Signer invalid or inactive for current epoch"` | Exact error string, check at VA 0x3355790 | Agent: signer epoch |
| `override_consensus_rpc_c_signers.json` | Exact file path string | Binary strings |
| `self_delegation_requirement` | Exact field name, `"Self delegation insufficient"` error | Binary strings |
| Source: `l1/src/staking.rs` | Debug path embedded | Binary strings |

### RPC Backfill

| Implementation Choice | Binary Evidence | Location |
|----------------------|-----------------|----------|
| `RpcRequestContent::BlocksAndTxs {after_round, n, last_block_hash}` | Field names in serde strings | Binary strings |
| Round-robin peer selection | `RpcRoundRobin` error string | Binary strings |
| 9 ClientBlock validation checks | 9 distinct error variants (ClientBlockQcRound through CommitProofConsecutive) | VA 0x6600fa |
| `"too many blocks to request"` safeguard | Exact string | Binary strings |
| `"bootstrap ended past original block, is peer malicious?"` | Exact string | Binary strings |
| `"forward_client_blocks no more blocks from reader"` | Exact string | Binary strings |
| Source: `node/src/consensus/rpc.rs` | Debug path embedded | Binary strings |

---

*This document is derived entirely from reverse engineering. It is not official Hyperliquid documentation. Accuracy is best-effort based on binary analysis of the deployed validator binary.*

#protocol #specification #confirmed
