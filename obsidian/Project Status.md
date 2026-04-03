# Project Status

Current state of the hyperliquid-rust reimplementation as of 2026-04-02.

## Stats
- **13 crates**, ~18,000+ lines of Rust, **402 tests passing**
- 29 Obsidian docs
- 3 repos: lastdotnet/hyperliquid-rust, lastdotnet/hlz, lastdotnet/foundry-hl
- Deployed on hyperscan (Tailscale: `100.85.232.55`, 64 cores, 755GB RAM)
- GhidraMCP running for interactive binary RE

## Architecture

```
Developer Tools:
  hlz (trading CLI)  →  L1 API (:3001)
  forge/cast         →  EVM RPC (:8545)
  GhidraMCP          →  Ghidra (:8080)

hlx (node):
  ├── devnet          — local chain with mock services
  ├── devnet --fork   — mainnet state bootstrap (2.1M users, $3B)
  ├── follow          — replay blocks, compare hashes
  ├── gossip          — connect to mainnet peers
  └── run-non-validator — live sync (WIP)
```

## Working Systems

### Local Devnet (hl-devnet)
- `hlx devnet --enable-evm --oracle-mode live --fork mainnet_state.rmp`
- Mock oracle (static/synthetic/live), bridge, broadcaster
- Block producer (70ms blocks), 110 seeded orders across 11 assets
- HTTP API: 30+ /info query types, /exchange, /devnet/deposit
- EVM JSON-RPC: eth_call, eth_getStorageAt, eth_getCode, debug_traceCall
- HyperCore RPC: hypercore_getPosition, getOraclePrice, getBalance
- WebSocket: allMids, l2Book, bbo, trades subscriptions
- Feature flags: `dev` (cheatcodes), `--no-default-features` (production)

### Matching Engine (hl-engine)
- Order books with BTreeMap + VecDeque + O(1) OID cancel
- 80 action types handled with real state mutations
- Service layers: staking, vaults, sub-accounts, BOLE lending, outcomes, agents
- Funding settlement (EMA premium, **CONFIRMED 1h period** = 3600s, not 8h)
- Security audit: 30 findings, all criticals fixed

### State Hash
- **Algorithm CRACKED**: rmp_serde → blake3 XOF (2048B) → paddw u16 → **SHA-256** (32B)
- **EVM format CRACKED**: raw byte concat (92B accounts, 84B storage, variable contracts)
- **11 L1 + 3 EVM = 14 total accumulators** confirmed with data structures
- **Heartbeat wire layout FULLY LIFTED** (2026-04-02):
  - `ConciseLtHashes` = {accounts_hash, contracts_hash, storage_hash}
  - Each `ConciseLtHash` = {hash_concise: **32B SHA-256**, n_elements: u64, n_bytes: u64}
  - **hash_concise is SHA-256 of full LtHash16**, NOT the raw 2048B accumulator
  - Full accumulators only available from ABCI state snapshots
- **HeartbeatSnapshot** = {ValidatorSetSnapshot{stakes, jailed_validators}, evm_block_number, ConciseLtHashes}
- **VoteAppHash** = {height, appHash, signatures} — tracked in `app_hash_vote_tracker`, quorum at `quorum_app_hashes`
- **Remaining**: exact L1 element field ordering for hash verification

### Gossip Protocol — FULLY CRACKED (2026-04-02)
- **Wire format**: `[u32 BE body_len][u8 kind=0x01][LZ4-compressed bincode body]`
- **Tag 27 = MsgConcise** (8 inner variants): Tx, Block, Vote, Timeout, Tc, BlocksAndTxs, Heartbeat, HeartbeatAck
- **Tag 28 = OutMsgConcise** (9+ inner variants): Block, ClientBlock, VoteJailBundle, Timeout, Tc, Heartbeat, NextProposer, Tx, RpcRequest
- **Mid 0-4** = consensus flow direction: in, execution state, out, status, round advance
- **MsgConcise outer struct**: {source, msg (mid-tagged), prev_round, reason}
- **Full type path**: `node::consensus::rpc::RpcUpdate`
- **6-layer send path** fully traced from field serializers → TCP frame write
- **Live validated**: 20-frame remote peer capture, all parse correctly with confirmed enums
- Detailed function addresses, jump tables, and caller chains in [[Gossip Protocol]]

### Testnet Validator “HypurrLiquid” (2026-04-02)
- **Registered**: validator `0x6db9d0c63aae7f6e2d143e9f592b622b65213689`, signer `0x929e...bbeb`
- **Stake**: 17,498 HYPE (rank #83, need top 50 for active set)
- **Status**: jailed, not active — need ~156K HYPE to enter active set
- **Running on**: nest (home Linux box, `100.118.30.68`, public IP `47.54.5.67`)
- **Binary patch**: NOPed `node_ips` assertion at config.rs:60 (2 conditional jumps)
- **Solved issues**: process detection (tmux), firewall_ips (file_mod_time_tracker/), IP override, GPG verification, port forwarding (UniFi WAN2)
- **Blocker**: `CSignerAction::unjailSelf` requires signer in active epoch set (top 50 by stake)
- **”Unbounded send error”** during non-validator→validator transition is **expected behavior**

### Validator Admin Surfaces (from binary RE)
- **80 action variants** in main Action enum
- **CSignerAction** (3 variants): unjailSelf, jailSelf, jailSelfIsolated
- **CValidatorAction** (16 variants): asset registration, oracle, margins, fees, governance
- **Bridge actions**: VoteEthDeposit, ValidatorSignWithdrawal, SignValidatorSetUpdate, VoteEthFinalized*
- **VoteAppHash**: {height, appHash, signatures} → `app_hash_vote_tracker` → `quorum_app_hashes`
- **Override path**: `/override_consensus_rpc_c_signers.json` (local only)
    - `round_to_jailed_validators` → `FUN_043b91a0`
- **New caller-edge map**:
  - `FUN_043b8330` <- `FUN_043b7620`, `FUN_04593d40`
  - `FUN_043b8670` <- `FUN_03ff8540`
  - `FUN_043b8dc0` <- `FUN_04593650`
  - `FUN_043b8f70` <- `FUN_0459ece0`
- **New structural lift from disassembly**:
  - `FUN_043b8330` reads `proposer`, `round`, `tx_hashes`, `time`, literal `qc`, literal `tc`, `hash`
  - `FUN_043b8670` reads `prev_round`, `round`, `reason`
  - `FUN_043b8dc0` reads `round`, `time`, `txs`, `proof_len`, `txs_len`
  - `FUN_043b8f70` reads literal inner `Qc` / `Tc`, then `proposer`, `next_proposer`, `suspect`, `last_vote_round`
  - local `objdump` now confirms the exact anchors for `FUN_043b8330` at `proposer` (`0x1e0d10`), `round` (`0x65fa1b`), `tx_hashes` (`0x66048f`), `time` (`0x1dfb4c`), `qc` / `tc` (`0x660498` / `0x66049a`), and `hash` (`0x1dfb74`)
  - local `objdump` also shows `FUN_043b8f70` literally writing `Qc` (`0x6351`) vs `Tc` (`0x6354`) before the rest of that payload
- **Implication**: the deployed binary really does separate heartbeat/snapshot serialization from a neighboring concise/consensus family, which matches the live 4001 result: the traffic we see locally looks closer to the `0x043b83..0x043b90..` family than to the heartbeat snapshot path
- **Confirmed outer mapping**: live peer `tag 27` is `MsgConcise` and `tag 28` is `OutMsgConcise`, both under `node::consensus::rpc::RpcUpdate`. The remaining decode gap is the exact inner payload layout within each branch, not the outer family identity.
- Heartbeats contain `validator_set_snapshot`, `evm_block_number`, and `concise_lt_hashes`
- **Correction**: traced heartbeat `concise_lt_hashes` serializer currently shows only `accounts_hash`, `contracts_hash`, `storage_hash`
- **Current gap**: longer non-validator localhost sweeps still showed no heartbeat/app-hash field-name markers on that loopback path. Current read is that real heartbeat/app-hash payloads require validator-mode traffic or a direct validator/sentry peer.
- VoteAppHash every 2000 blocks, 3 validators agree

### HyperEVM (hl-evm)
- revm 36 with Cancun spec, 30M gas limit
- L1StateProvider precompiles (7 query types)
- CoreWriter (15 action types)
- ForkDb: lazy EVM state from remote RPC
- Debug: debug_traceCall, eth_getStorageAt

### foundry-hl
- `HyperCore.sol` — Solidity library wrapping hypercore_* RPC via vm.rpc()
- `CoreWriter.sol` — interface for 0x3333...3333 (15 actions)
- Example forge tests with cross-layer L1↔EVM assertions
- `forge test --fork-url http://127.0.0.1:8545`

### Chain Follower
- `hlx follow --data-path ... --snapshot ...`
- Replays blocks through engine, extracts VoteAppHash
- Compares our hash against mainnet — divergence reports saved
- Currently mismatching (L1 accumulator starts at zero, not genesis value)

### hlz (trading CLI)
- Fork of dzmbs/hlz with `--chain local` support
- Connects to hlx devnet on 127.0.0.1:3001 (HTTP) and :3002 (WS)
- `hlz perps`, `hlz price`, `hlz book BTC`, `hlz portfolio`

## Security Findings (CONFIRMED 2026-04-02)

### Critical (4)
1. **Broadcaster centralization** -- single broadcaster controls action ordering, inclusion, and can extract MEV
2. **Bridge InvalidateWithdrawals** -- validators can cancel ANY pending withdrawal during dispute period, no escape hatch
3. **No escape hatch** -- users cannot force-withdraw if validator set becomes adversarial
4. **Broadcaster MEV** -- broadcaster sees all pending transactions before inclusion, can front-run/sandwich/censor

### High (6)
5. Bridge validator set modifiable via ModifyBroadcaster (instant, no timelock)
6. Single oracle_updater (broadcaster) controls all perp prices
7. Validator set update on Ethereum side can be changed with hot key quorum
8. No slashing for Byzantine behavior (only jailing)
9. No rate limiting on bridge invalidation actions
10. Validator rewards concentrated in top stakers

### Confirmed Incidents
- **$38M+ total confirmed losses** across 5 documented incidents
- Most related to oracle manipulation and liquidation cascades

### INFERRED to CONFIRMED Status Updates (2026-04-02)
| Finding | Previous | Updated |
|---------|----------|---------|
| Oracle = stake-weighted median | INFERRED | **CONFIRMED** |
| Funding = 3600s settlement | INFERRED | **CONFIRMED** |
| ADL = ROE ranking | INFERRED | **CONFIRMED** |
| BOLE = 3 liquidation types with 0.8 threshold | INFERRED | **CONFIRMED** |
| HIP-3 stale mark = 10s window | INFERRED | **CONFIRMED** |
| Epoch = time-based (epoch_duration_seconds) | INFERRED (was "round-based") | **CONFIRMED** (corrected) |
| Bridge signing = consensus signing keys | INFERRED | **CONFIRMED** |
| begin_block = 9 effects | INFERRED (was 5) | **CONFIRMED** (VA 0x01e748e0) |

## Gap Analysis

### Level 1: Passive Observer (95%)
| Component | Status |
|-----------|--------|
| Gossip connect + stream | ✅ Connected to mainnet, receiving data |
| Block parsing | ✅ 99.96% of 80 action types |
| State bootstrap | ✅ 2.1M users, $3B, 314 books |
| Matching engine | ✅ With all service layers |
| State hash algorithm | ✅ SHA-256 + blake3 + rmp confirmed |
| **Gossip bootstrap framing** | ⚠️ Mixed/TBD — need exact greeting/state-transfer decoder |
| **L1 hash accumulator** | ⚠️ Need starting value from heartbeat |
| **Exact hash serialization** | ⚠️ Field ordering TBD via differential testing |

### Level 2: Full Non-Validator (80%)
| Component | Status |
|-----------|--------|
| /info API | ✅ 30+ query types |
| EVM RPC | ✅ Full JSON-RPC + precompiles + debug |
| Local devnet | ✅ Fork mode + live oracle |
| Fork from mainnet | ✅ (memory-optimized bootstrap) |
| **All action side-effects** | ⚠️ 80 handled, most wired with real state |
| **app_hash exact match** | ❌ Need L1 accumulator from gossip/heartbeat |

### Level 3: Validator (30%)
| Component | Status |
|-----------|--------|
| Consensus voting | ❌ Not implemented |
| Block proposal | ❌ Not implemented |
| Heartbeat send/receive | ❌ Need exact greeting/bootstrap framing and heartbeat extraction |
| Epoch transitions | ❌ Stub |

## Key Binary RE Findings

### Hash Mechanism (CONFIRMED)
```
Each state mutation → rmp_serde::to_vec() → blake3::Hasher (streaming Write)
  → blake3 XOF (2048 bytes) → SSE2 paddw u16 wrapping-add → LtHash16 checksum
  → SHA-256(checksum) → 32-byte app_hash
```

### Key Addresses (Ghidra)
| Function | VA | Description |
|----------|-----|------------|
| Main Exchange dispatcher | `0x0272a320` | >500KB, all action dispatch |
| Blake3 hasher write | `0x04e03010` | LtHash element hashing |
| SSE2 paddw (u16 add) | `0x04e05786` | LtHash insert |
| LtHash insert processing | `0x04e0ccf0` | Full insert with sort |
| ConciseLtHash serde pair | `0x043db480` / `0x04387640` | `n_elements`, `n_bytes`, `hash_concise` |
| HeartbeatSnapshot serde pair | `0x0438fcc0` / `0x043b9a10` | `validator_set_snapshot`, `evm_block_number`, `concise_lt_hashes` |
| Greeting builder | `0x02767ee0` | Builds ABCI greeting |
| Peer rejection logic | `0x75fe5e` | maybe_reject_peer_reason |

### 2026-04-01 Ghidra Sweep

#### Gossip cluster (new string/xref anchors)
- Peer verification cluster:
  - `performing checks on stream` → `0x004a2989`
  - `no quorum yet` → `0x004a29c2`
  - `peer is already connected` → `0x004a2a4b`
  - `sending abci_state` → `0x004a2aef`
  - `sending evm kvs` → `0x004a2b48`
  - `max peers reached` → `0x004a2c14`, `0x00760c44`
  - `not validator or sentry` → `0x004a2c25`, `0x00760c55`
  - `abci state request rate limited` → `0x004a2c3c`, `0x00760c6c`
  - `verify_rpc` → `0x004a2ca1`
- Health / stream cluster:
  - `local height is from stale hardfork` → `0x00474335`
  - `local height is too far behind` → `0x00474358`
  - `forward_client_blocks` → `0x0076d9d9`
  - `gossip stream msg` → `0x0076da09`

#### EVM snapshot / hash cluster
- `EvmKvsMsg` → `0x0047447d`, `0x00492df4`, `0x0075ae05`, `0x00769063`, `0x0076da5b`
- `InvalidMagic` → `0x00492e0a`, `0x0063289c`
- `UnsupportedVersion` → `0x00492e16`, `0x004ac514`, `0x006328a8`
- `InvalidLength` → `0x00492dfd`, `0x0063288f`
- Heartbeat/app-hash field-name cluster:
  - `validator_set_snapshot` → `0x0076055d`, `0x00761eda`
  - `concise_lt_hashes` → `0x00760573`
  - `validator_to_last_msg_round` → `0x0076059e`
  - `random_id` → `0x007605b9`
  - `evm_block_number` → `0x002de290`
  - `hash_concise` → `0x0075fb55`
  - `n_elements` → `0x0075fb37`
  - `n_bytes` → `0x0075fb41`
  - Additional type-name anchors:
    - `ConciseLtHashes` → `0x0075fb1a`
    - `EvmTxHash` → `0x0075fb11`
    - `SharePx` → `0x0075fb29`
    - `UsdcPx` → `0x0075fb31`

#### Serializer path update
- `FUN_0438fcc0` serializes:
  - `validator_set_snapshot`
  - `evm_block_number`
  - `concise_lt_hashes`
- `FUN_043b9a10` deserializes the same three-field `HeartbeatSnapshot`
- `FUN_045a0520 -> FUN_04381a00` serializes exactly:
  - `accounts_hash`
  - `contracts_hash`
  - `storage_hash`
- `FUN_043db480` serializes a `ConciseLtHash` object:
  - `n_elements`
  - `n_bytes`
  - `hash_concise`
- `FUN_04387640` deserializes the same `ConciseLtHash`
- `FUN_0438fa90` / `FUN_043baac0` are a parent serde pair above `HeartbeatSnapshot`
  - confirmed fields: `validator`, `round`, `snapshot`, `validator_to_last_msg_round`, `random_id`
  - `snapshot` is the nested child routed through `FUN_0438fcc0` / `FUN_043b9a10`
- `FUN_04390050` is likely a text/debug/display formatter rather than the live heartbeat serializer:
  - caller `FUN_047c23c0` emits punctuation/comma/braces around it
- Practical implication:
  - the L1 `hash_concise` accumulator is still not confirmed on the traced heartbeat wire path
  - `SharePx` exists in adjacent type helpers, but is not yet confirmed as a serialized heartbeat field

### Wire Protocol
- Greeting: 6-byte bincode-fork `TcpGreeting` enum with 2 variants (`send_abci`, `broadcast_group`), framed with u32 BE length prefix
- Chain enum: Local=0, Sandbox=1, Testnet=2, Mainnet=3
- Connection requires known peer status -- server-side `connection_checks` verify IP is known validator/sentry, times out after 5s
- Post-greeting: full ABCI state + EVM KVs, but exact outer encoding is still mixed/TBD
- 4001 stream outer frame: `[u32 BE body_len][u8 kind][body]`, with observed `kind = 0x01`
- Many bodies: LZ4 size-prepended
- Inner payloads: not direct `ReplicaCmd` msgpack in the first live samples; at least some control frames look like prefixed varint-bincode
- Heartbeats: traced serializer path contains `validator_set_snapshot`, `evm_block_number`, and `concise_lt_hashes`
- Current correction: traced `concise_lt_hashes` path carries the three EVM hash accumulators; L1 `hash_concise` transport is still unresolved

### Durability Model
- L1 state: in-memory, periodic msgpack snapshots (~1.1GB every ~10K blocks)
- EVM state: RocksDB (`EvmStateDbHome`)
- Blocks: NDJSON (`replica_cmds/`) — serves as write-ahead log
- Recovery: load snapshot + replay replica_cmds from snapshot height

## Blockers & Next Steps

### Critical Path: Crack the L1 Hash
1. **Map the small prefixes in front of inner varint-bincode control/consensus frames**
2. **Decode the actual greeting/bootstrap framing** to parse gossip greeting and EVM KVs precisely
3. **Find the live transport location for the L1 `hash_concise` accumulator**
4. With the real L1 accumulator → verify our hash format via chain follower
4. Iterate on Settlement struct field ordering until hashes match

### Secondary
- Determine exactly where `rmp` ends and any custom bootstrap framing begins
- Implement consensus voting (needs 10K HYPE testnet stake)
- Wire foundry-hl for full Solidity testing workflow
- Build the chain-following auto-heal mode

## Links
- [[ABCI State Machine]] — block processing pipeline
- [[Gossip Protocol]] — networking details + function addresses
- [[App Hash]] — hash structure and verification
- [[LtHash Serialization]] — confirmed element formats
- [[HyperEVM]] — EVM integration
- [[Devnet]] — local development network
- [[Exchange State]] — 57-field god struct

#project #status #roadmap
