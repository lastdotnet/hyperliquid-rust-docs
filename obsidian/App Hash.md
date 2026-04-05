# App Hash

The state hash used by [[HyperBFT]] consensus to verify all validators agree on state.

## Algorithm (CONFIRMED 2026-04-04)

```
Action Response → rmp_serde::to_vec_named(response) → blake3::Hasher (streaming Write)
    → blake3 XOF (2048 bytes) → interpret as 1024 × u16 LE
    → SSE2 paddw wrapping-add to per-category LtHash16 accumulator
    → accumulate n_elements, n_bytes

Final app_hash (32 bytes):
    L1_combined  = paddw(all 11 L1 category accumulators)
    EVM_combined = paddw(all 3 EVM category accumulators)
    app_hash     = first_16(SHA-256(L1_combined)) || first_16(SHA-256(EVM_combined))
```

**CONFIRMED 2026-04-04**: app_hash is NOT a single SHA-256 of all 14 combined. It is the concatenation of the first 16 bytes of each half's SHA-256 digest. Verified from node log at height 536290000 where L1 portions matched but EVM differed.

**Key confirmations from binary RE:**
- Compression: **SHA-256** (sha2 0.10.8) — NOT SHA3, NOT Keccak
- XOF: **BLAKE3** 1.7.0 with SSE2/AVX2 acceleration
- Serialization: **rmp_serde** writes directly into blake3 Hasher via Write trait
- SIMD: **SSE2 paddw** for u16 wrapping-add at `0x04e05786`

## Heartbeat `concise_lt_hashes` (CONFIRMED 2026-04-02)

**CONFIRMED**: Heartbeat `concise_lt_hashes` contains **only EVM hashes** (accounts_hash, contracts_hash, storage_hash). The L1 accumulators (Na, CValidator, Settlement, etc.) are **NOT** in heartbeats -- they are always rebuilt from state.

```rust
struct ConciseLtHashes {
    accounts_hash: ConciseLtHash,   // EVM accounts
    contracts_hash: ConciseLtHash,  // EVM contracts
    storage_hash: ConciseLtHash,    // EVM storage
}

struct ConciseLtHash {
    hash_concise: [u8; 32],     // SHA-256 digest of the full LtHash16 checksum
    n_elements: u64,
    n_bytes: u64,
}
```

**Key corrections:**
- The heartbeat wire format carries a 32-byte `hash_concise` digest, not the full 2048-byte LtHash16 accumulator
- Heartbeats alone are NOT enough to reconstruct the full accumulator state
- L1 hash categories (11 types) are computed from state, not transmitted in heartbeats

**Traced wire path:**
- `FUN_0438fcc0` serializes `HeartbeatSnapshot { validator_set_snapshot, evm_block_number, concise_lt_hashes }`
- `FUN_043b9a10` deserializes the same `HeartbeatSnapshot`
- the `concise_lt_hashes` field itself is serialized through `FUN_045a0520`
- `FUN_045a0520` calls `FUN_04381a00`
- `FUN_04381a00` serializes exactly:
  - `accounts_hash`
  - `contracts_hash`
  - `storage_hash`

**Adjacent but NOT confirmed on the heartbeat serializer path:**
- `hash_concise` has its own `ConciseLtHash` serde pair:
  - serializer `FUN_043db480`
  - deserializer `FUN_04387640`
- `SharePx` exists in the binary (`0x0075fb29`) and has a tiny label helper at `FUN_04381c60`
- `EvmTxHash` exists in the same type-name cluster (`0x0075fb11`)
- None of those have been seen on the traced `FUN_043b9a10 -> FUN_045a0520 -> FUN_04381a00` heartbeat path yet

## 11 L1 Hash Categories (hash_concise enum, CONFIRMED 2026-04-01)

| # | Category | Shares code path with |
|---|----------|-----------------------|
| 0 | CValidator | CSigner (entries [0,1]) |
| 1 | CSigner | CValidator (entries [0,1]) |
| 2 | Na | LiquidatedCross (entries [2,3]) |
| 3 | LiquidatedCross | Na (entries [2,3]) |
| 4 | LiquidatedIsolated | (unique path) |
| 5 | Settlement | (unique path) |
| 6 | NetChildVaultPositions | (unique path) |
| 7 | SpotDustConversion | BoleMarketLiquidation (entries [7,8]) |
| 8 | BoleMarketLiquidation | SpotDustConversion (entries [7,8]) |
| 9 | BoleBackstopLiquidation | BolePartialLiquidation (entries [9,10]) |
| 10 | BolePartialLiquidation | BoleBackstopLiquidation (entries [9,10]) |

String offsets in binary: `0x65fb61` through `0x65fc08`

### RespHash dispatch architecture (from RE of vaddr 0x205a300)

The dispatch function takes a response type enum (0-19) and serializes the
response struct via rmp_serde named-map format. All 11 categories use the SAME
response enum struct type; the category only determines which accumulator
receives the element. The 20-entry jump table at vaddr 0x37fe2c maps response
types to 14 unique code paths. Shared paths (same struct layout) indicate
response types that differ only in which LtHash16 accumulator they target.

### 2026-04-05 Serializer Clarifications

- G-family hybrid actions do **not** hash a 3-field payload on success. Success switches to the same 1-field `{"status":"success"}` map used by the H-family status path.
- The 3-field `{"status":"err","success":false,"error":"..."}` payload is error-only.
- Fills hash through a separate 18-field payload; API wrapper fields like `coin` and `feeToken` are not members of that hashed fill payload.
- The current local open lane is exact fill value derivation, not basic payload membership.

Response field names (from binary rodata at 0x65f9cb):
`sz isz executedSz executedNtl minutes reduceOnly randomize timestamp
 running twapId error status filled totalSz avgPx oid cloid resting
 waitingForFill waitingForTrigger`

Each serialization path uses a 256-entry vtable of function pointers (2048 bytes
in data segment at 0x5006xxc8), with vtable slot [10] being the only
type-differentiating entry (the variant-specific payload serializer).

## EVM Element Formats (CONFIRMED from lt_hashes.json)

### accounts_hash — 92 bytes per element
```
address(20B) || balance(32B BE) || nonce(8B BE) || code_hash(32B)
```

### storage_hash — 84 bytes per element
```
address(20B) || slot(32B) || value(32B)
```

### contracts_hash — variable bytes per element
```
address(20B) || code(variable bytecode)
```

## VoteAppHash

- Emitted every **2000 blocks** as a governance action
- **3 validators** vote the same hash (deterministic)
- Appears **~150 blocks** after the checkpoint height
- Format: `{"type":"voteAppHash","height":N,"appHash":"0x...","signature":{r,s,v}}`

## Known Targets

| Height | App Hash | Source |
|--------|----------|--------|
| 941590000 | `0x50002b054fd6038d...` | VoteAppHash in replica_cmds |
| 941660000 | `0xeaa14930e8b0fdd6...` | VoteAppHash + exact state/EVM match |
| 942360000 | `0xc0bf9918f189827b...` | hl-node log on hyperscan |

## Accumulator Properties

- L1 hash is a **running accumulator from genesis** — NOT stored in periodic snapshots
- EVM hashes ARE stored in `lt_hashes.json` per checkpoint
- LtHash is **order-independent** (lattice property): H(A∪B) = H(A) + H(B)
- Insert: checksum += XOF(element), n_elements++, n_bytes += len
- Remove: checksum -= XOF(element), n_elements--, n_bytes -= len

## What's Cracked vs Remaining

### CRACKED
- Full algorithm pipeline (rmp -> blake3 -> paddw -> SHA-256)
- EVM element serialization (raw byte concat, exact sizes)
- All 11 L1 categories identified (CValidator through BolePartialLiquidation)
- ConciseLtHash core layout and heartbeat field names
- Wire-level correction: `hash_concise` is a 32-byte digest, not a 2048-byte LtHash buffer
- Heartbeat `concise_lt_hashes` serializer path for the EVM triplet
- HeartbeatSnapshot wire format
- VoteAppHash timing and format
- Key function addresses in binary
- **RespHash dispatch architecture** -- 20-entry jump table at vaddr 0x37fe2c, dispatch function at 0x205a300
- **ALL categories use the SAME response enum struct** -- serialized via rmp_serde compact format
- **256-entry serialization vtable** per response type -- only slot [10] differs (variant payload)
- **14 LtHash16 accumulators** stored contiguously at 0x800-byte intervals (0x50066c8 to 0x500cec8)
- **Response field names** from binary rodata: sz, isz, executedSz, executedNtl, minutes, reduceOnly, randomize, timestamp, running, twapId, error, status, filled, totalSz, avgPx, oid, cloid, resting, waitingForFill, waitingForTrigger
- **Category pairing** -- response types that share serialization code: [0,1], [2,3], [7,8], [9,10], [12,13,14]
- **COMPLETE per-struct field serialization order** -- extracted from testnet binary Serialize impls (see "Response Struct Field Serialization Order" section below)
- **Additional fields discovered**: triggerCondition, isTrigger, isPositionTpsl, startPosition, feeToken, triggered, coin, user, side, limitPx, orderType, origSz

### REMAINING
- **L1 accumulator starting value** -- need from gossip heartbeat or replay from genesis
- **Where the L1 `hash_concise` travels on the wire** -- the traced heartbeat `concise_lt_hashes` path currently shows only EVM hashes
- **Recovering full accumulators from heartbeats** -- not possible from current wire evidence because only the 32-byte digest is transmitted
- **`SharePx` / `EvmTxHash` participation** -- strings/type helpers exist, but they are not yet on the traced heartbeat serializer path

## Response Struct Field Serialization Order (EXTRACTED 2026-04-01, testnet binary)

Extracted from testnet binary `/tmp/target_bin_analysis` on hyperscan.
Function `compute_resps_hash` at VA `0x2355860` dispatches on 20 response types
via jump table at VA `0x3aa3f8`. Types 3,4,5,6,17,18,19 have secondary sub-dispatch.
The serde Serialize impls produce the same field order for both JSON and rmp_serde.

### OrderStatus ENUM (externally tagged serde enum)

| Variant | Inner fields (in order) |
|---------|------------------------|
| `"filled"` | totalSz, avgPx, oid |
| `"error"` | (plain string, no inner fields) |
| `"waitingForFill"` | oid |
| `"waitingForTrigger"` | oid |
| `"resting"` | oid, cloid |

Serialize function at `0x4a75120` (3238 bytes, handles all variants).

### RestingOrder (full order on book, CORRECTED to 15 fields 2026-04-02)

```
coin, side, limitPx, sz, timestamp, triggerCondition, isTrigger,
triggerPx, isPositionTpsl, reduceOnly, orderType, origSz,
tif, cloid, children
```
Serialize function at `0x4a98d30`. **CORRECTED**: 15 fields (was 12). Added `tif` (time-in-force), `cloid` (client order ID), and `children` (child TP/SL orders).

### FillResponse (CORRECTED to 24 fields 2026-04-02)
The FillResponse is the most complex response struct. On mainnet it has **24 fields** (not the 10 or 21 previously documented). Key correction: the response uses an **externally-tagged OrderStatus enum** (not a flat struct), so the JSON output looks like `{"filled": {"totalSz": 1.0, ...}}` rather than `{"filled": true, "totalSz": 1.0, ...}`.

### CancelResponse (CORRECTED 2026-04-02)
Cancel response format is `{"status": "success"}` -- a StatusResult with a status string field. NOT `{"success": null}` as previously assumed.

### TwapSliceFill (TwapOrder response, type 9)

```
coin, user, side, sz, executedSz, executedNtl, minutes,
reduceOnly, randomize, timestamp
```
Serialize function at `0x4a701a0` (10 fields).

### TwapState (TwapCancel response, type 10)

```
running, twapId, error
```
Serialize function at `0x4a70410`.

### Fill/Trade (execution fill)

```
coin, sz, side, startPosition, hash, fee, tid, feeToken, twapId
```
Serialize function at `0x4a47ec0` (9 fields).

### SimpleResult (success/error)

```
success, error
```
Serialize function at `0x4a707f0`.

### StatusResult (single status)

```
status
```
Serialize function at `0x4a70a60`.

### OrderBase (basic order fields)

```
side, sz, reduceOnly
```
Serialize function at `0x4a80610`.

### OrderWithTiming (order + timing)

```
side, sz, reduceOnly, randomize, timestamp
```
Serialize function at `0x4a82640`.

### TriggerParams

```
triggerPx, tpsl
```
Serialize function at `0x4a8d2d0`.

### TriggeredFill

```
triggered, filled
```
Serialize function at `0x4a99070`.

### Type-to-Struct Dispatch (testnet binary)

| Type | Jump target | Struct | Notes |
|------|------------|--------|-------|
| 0 | `0x23558bb` | Array (CValidator) | Complex array processing |
| 1 | `0x2355d62` | Error path | Loads error/panic string |
| 2 | `0x2355d55` | Error path | Similar to type 1 |
| 3 | `0x235597b` | OrderStatus (sub-dispatch) | Secondary table at `0x3aa430` |
| 4 | `0x2355913` | CancelResponse (sub-dispatch) | Secondary table at `0x3aa420` |
| 5 | `0x235593a` | ModifyResponse (sub-dispatch) | Secondary table at `0x3aa410` |
| 6 | `0x2355940` | BatchModifyResponse | Shares table with type 5 |
| 7 | `0x2355da0` | Error path | |
| 8 | `0x2355d7c` | Error path | |
| 9 | `0x235934d` | TwapSliceFill | Own serialize path |
| 10 | `0x2355909` | Error path | |
| 11 | `0x2355d7e` | Error path | |
| 12 | `0x2355d5a` | Error path | |
| 13 | `0x2358b52` | Complex (RestingOrder) | Own serialize path |
| 14 | `0x235596f` | Error path | |
| 15 | `0x2355da4` | Error path | |
| 16 | `0x2355d92` | Error path | |
| 17 | `0x2355986` | Sub-dispatch | Shares table with type 3 |
| 18 | `0x23559d1` | Sub-dispatch | Own table |
| 19 | `0x23559e9` | Sub-dispatch | TwapSliceFill variant |

**Note:** Types labeled "Error path" still perform response processing but
converge to the common serialization path at `0x2355fbc` (PLT call to
rmp_serde). The actual serialization uses the Serialize trait vtable dispatch,
so the field order is determined by the struct definition, not the dispatch type.

## Key Function Addresses

| Function | VA | Purpose |
|----------|-----|---------|
| RespHash dispatch | `0x0205a300` | Main dispatch: sil=category(0-19), jump table at `0x37fe2c` |
| RespHash caller (add) | `0x02076fb3` | Calls dispatch from block processing (add element) |
| RespHash caller (remove) | `0x0207724d` | Calls dispatch from block processing (remove element) |
| blake3 hasher write | `0x04e03010` | Streaming hash input |
| SSE2 paddw | `0x04e05786` | u16 wrapping-add |
| LtHash insert | `0x04e0ccf0` | Full insert logic |
| ConciseLtHash serde pair | `0x043db480` / `0x04387640` | `n_elements`, `n_bytes`, `hash_concise` |
| HeartbeatSnapshot serde pair | `0x0438fcc0` / `0x043b9a10` | `validator_set_snapshot`, `evm_block_number`, `concise_lt_hashes` |
| Greeting builder | `0x02767ee0` | ABCI greeting |
| Per-event hash (0x22a spacing) | `0x047f6cb0` | 15 response-type element hashers at 0x22a intervals |
| Per-event hash helper | `0x047f78b0` | rmp_serde serialize + blake3 XOF + LtHash accumulate |

## 2026-04-01 Ghidra Sweep

- Heartbeat field-name anchors:
  - `validator_set_snapshot` → `0x0076055d`, `0x00761eda`
  - `concise_lt_hashes` → `0x00760573`
  - `validator_to_last_msg_round` → `0x0076059e`
  - `random_id` → `0x007605b9`
  - `evm_block_number` → `0x002de290`
  - `hash_concise` → `0x0075fb55`
  - `n_elements` → `0x0075fb37`
  - `n_bytes` → `0x0075fb41`
- `share_px` / `sharePx` produced no string hit via the Ghidra HTTP bridge in this pass.
- Practical implication: `hash_concise`, `n_elements`, and `n_bytes` are strong confidence fields; `share_px` should stay marked as likely-but-unconfirmed until a better xref/decompile path confirms its serialized name and hash participation.

### Serializer path corrections

- `FUN_0438fcc0` serializes:
  - `validator_set_snapshot`
  - `evm_block_number`
  - `concise_lt_hashes`
- `FUN_043b9a10` deserializes the same three-field `HeartbeatSnapshot`
- `FUN_045a0520` calls `FUN_04381a00`, which serializes exactly:
  - `accounts_hash`
  - `contracts_hash`
  - `storage_hash`
- `FUN_043db480` serializes `ConciseLtHash { n_elements, n_bytes, hash_concise }`
- `FUN_04387640` deserializes the same `ConciseLtHash`
- `FUN_0438fa90` / `FUN_043baac0` are a parent serde pair above `HeartbeatSnapshot`
  - confirmed fields: `validator`, `round`, `snapshot`, `validator_to_last_msg_round`, `random_id`, `prev_round`, `reason`
  - `snapshot` is the nested child routed through `FUN_0438fcc0` / `FUN_043b9a10`
- `FUN_04390050` is likely a text/debug/display formatter, not the live heartbeat serializer:
  - its caller `FUN_047c23c0` emits punctuation/comma/braces around the object
- New string/type anchors in the same cluster:
  - `ConciseLtHashes` → `0x0075fb1a`
  - `SharePx` → `0x0075fb29`
  - `UsdcPx` → `0x0075fb31`
  - `EvmTxHash` → `0x0075fb11`
- Current best read:
  - the live heartbeat `concise_lt_hashes` path definitely carries the three EVM `ConciseLtHash` values
  - `hash_concise`, `SharePx`, and `EvmTxHash` are present in adjacent serializer/type clusters but not yet confirmed on that same wire path

## Links
- [[LtHash]] — algorithm details
- [[LtHash Serialization]] — element format details
- [[Gossip Protocol]] — how hashes are exchanged
- [[ABCI State Machine]] — when hashes are computed
- [[Exchange State]] — state being hashed

#hashing #consensus #confirmed
