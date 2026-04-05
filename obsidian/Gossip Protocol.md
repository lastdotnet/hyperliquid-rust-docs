# Gossip Protocol

Complete reverse engineering of the Hyperliquid gossip/P2P protocol.

## Ports
- **4001** — Block streaming (abci_stream)
- **4002** — Peer discovery / RPC verification

## TCP Greeting (Port 4001) — CONFIRMED 2026-04-04

### TcpGreeting (bincode-fork, chain-based variant index)

The variant index doubles as the chain identifier, encoded as **u32 BE**:

```
Testnet: variant 3 (BE), body = [00 01 00 00] → 8 bytes total
Mainnet: variant 2 (BE), body = [00 00 00]    → 7 bytes total
```

**Wire frame** (with kind byte):
```
[u32 BE body_len][u8 kind=0][greeting body]
```

**Server response** (6 bytes): `[u32 BE len=1][u8 kind=0][u8 chain_byte]`

After greeting exchange, bootstrap data streams: ~500MB ABCI state + ~4GB EVM RocksDB.

**Chain enum** (used in AbciStreamGreeting):
- Local = 0, Sandbox = 1, Testnet = 2, Mainnet = 3

## TCP Greeting (Port 4003) — Signing CONFIRMED 2026-04-04

### Authenticated Consensus Greeting

Port 4003 requires a signed greeting for the consensus stream. The signing formula has been **confirmed via GDB live capture**:

```
hash = keccak256("Hyperliquid Consensus Payload" + 0x00 + content_bytes)
sig  = secp256k1_sign_recoverable(hash, private_key)
```

**Critical**: A NULL byte (0x00) separates domain from content. Content is bincode-fork varint encoding, not msgpack, despite the `rmp_signable.rs` source file name.

**Signature details**:
- RFC 6979 deterministic nonces
- Low-S normalization (EIP-2)
- `v = recovery_id + 27` (Ethereum convention)
- r,s stored as 4 u64 LE limbs, each bincode-fork varint-encoded

**Wire format** (variable-sized; ~1.0-1.6KB in captured samples):
```
addr(20) + type(varint=5) + sig_r(4×u64_varint) + sig_s(4×u64_varint) + v(u8=27|28) + content(variable)
```

**Content layout** (byte layout mostly closed; a few middle counters still unlabeled):
- home-validator address (20B)
- round (varint u64)
- variant flag (u8)
- optional 32-byte ABCI extra when the variant requests it
- several additional counters/state values (mostly varints plus one raw 32-byte hash)
- `heartbeat_statuses` as a sorted `BTreeMap<Validator, HeartbeatData>`
- trailing mempool length varint (`0x00` in captured samples)

**Status**: Signing formula and most of the content byte layout are now closed from captured sign-input blobs plus Ghidra. The remaining open work is semantic labeling of a few middle counters, not basic serialization format.

### Connection Checks (CONFIRMED 2026-04-02)

Server-side `connection_checks` verify the connecting IP is a known validator/sentry. The check flow:
1. Server receives greeting
2. Server performs `connection_checks` — verifies IP against known peer list
3. Times out after **5 seconds** for unknown peers
4. Rejection reasons: `"not validator or sentry"`, `"max peers reached"`, `"abci state request rate limited"`, `"no quorum yet"`, `"peer is already connected"`
5. If accepted: `"performing checks on stream"` → `"verify_rpc"` → begin streaming

Found at: `0x474a9b`, `0x4a3093`, `0x75b105`, `0x76dc1d`

## Post-Greeting Flow

After the greeting exchange:

1. **Peer verifies us** — `"rpc connect"` / `"verify_rpc"` / `"performing checks on stream"`
   - May connect back to us on port 4002 to query `GossipStatus`
   - Rejection reasons: `"not validator or sentry"`, `"max peers reached"`, `"abci state request rate limited"`, `"no quorum yet"`, `"peer is already connected"`
   - Direct live trace on 2026-04-01 for a localhost session:
     - inbound 4001 greeting: `00000002000000`
     - server back-connects to port 4002
     - 4002 request: `000000010001`
     - 4002 response: `00000002000100`

2. **ABCI state transfer** — `"sending abci_state"` / `"serialized abci state for greeting"`
   - Length-prefixed blob; exact outer encoding is still being pinned down
   - Size: ~1.16GB at current state size
   - Contains: full Exchange state (57 fields) + concise_lt_hashes + app_hash

3. **EVM KV transfer** — `"sending evm kvs"` / `"EvmKvsMsg"`
   - Custom binary format with magic number + version
   - Size: ~4GB RocksDB snapshot
   - Contains: full EVM state (accounts, contracts, storage)

4. **Block streaming begins** — `"gossip stream msg"` / `"forward_client_blocks"`
   - Live stream framing is now confirmed as:
     - `[u32 BE body_len] [u8 kind] [body]`
     - observed `kind = 0x01` on all traced stream frames in this pass
   - Many live bodies are **LZ4 size-prepended blocks**:
     - raw body starts with little-endian decoded size
     - direct decode of captured bodies matches the prefixed size exactly
   - Example live writes from `hl-node`:
     - `sendto(..., 3776)` with prefix `00000ebb 01 ...`
     - `sendto(..., 430439)` with prefix `00069162 01 ...`
   - Practical implication:
     - the old `[len][payload]` assumption was off by one byte
     - the body itself starts after the `kind` byte
   - Current live decode result after LZ4:
     - **not** a direct `WireBlock`
     - **not** a direct heartbeat snapshot
     - **not** fixed-int `bincode`
     - stronger live result from longer localhost sweeps:
       - the observed family is dominated by an LZ4-decoded **concise control-frame** shape
       - first `72` bytes = `8` consecutive bincode-varint `u64` words (`0xfd + 8 LE bytes`)
       - byte `72` then starts a structured body:
         - `tag` = `27` or `28`
         - `addr` = 20 bytes
         - `round` = varint-encoded `u32` at body offset `21..26`
         - one-byte field at body offset `26` (`mid`) usually in `0..6`
         - small frames with `mid = 0` often carry another varint-encoded `u32` at body offset `27..32`
       - round increments by `1` across successive frames
       - known proposer `80f0cd23da5bf3a0101110cfd0f89c8a69a1384d` appears in the body address slot
     - practical implication:
       - the current loopback 4001 path looks much more like the binary `MsgConcise` / `OutMsgConcise` cluster than the older speculative `NetworkMessage` model
       - heartbeat/app-hash snapshots were still **not** seen on this live loopback path in this pass

5. **Heartbeats** — interspersed with blocks
   - Contains `HeartbeatSnapshot { validator_set_snapshot, evm_block_number, concise_lt_hashes }`
   - Confirmed serde pair:
     - serializer `FUN_0438fcc0`
     - deserializer `FUN_043b9a10`
   - Traced nested serialization:
     - `validator_set_snapshot`
     - `evm_block_number`
     - `concise_lt_hashes` via `FUN_045a0520 -> FUN_04381a00`
   - Parent wrapper above `HeartbeatSnapshot`:
     - serializer `FUN_0438fa90`
     - deserializer `FUN_043baac0`
     - confirmed fields: `validator`, `round`, `snapshot`, `validator_to_last_msg_round`, `random_id`
   - Current correction: the traced `concise_lt_hashes` wire path serializes `accounts_hash`, `contracts_hash`, `storage_hash`
   - The inner `ConciseLtHash` serde pair is:
     - serializer `FUN_043db480`
     - deserializer `FUN_04387640`
   - The L1 `hash_concise` accumulator is still present elsewhere in the binary, but not yet confirmed on this exact heartbeat serializer path

## Message Types (on the wire)

From binary RE at `0x76dc1d`:
```
TcpGreeting     — initial handshake
BlockTimeout    — consensus timeout
Heartbeat       — validator heartbeat (contains concise_lt_hashes)
HeartbeatAck    — response to heartbeat
ClientBlocks    — block data
Peers           — peer list
GossipStatus    — node status {initial_height, latest_height}
EvmKvsMsg       — EVM key-value snapshot
BlocksAndTxs    — combined block + tx data
```

### EvmKvsMsg Format
Has its own binary protocol:
- `FUN_02002c20` is a versioned parser for `EvmKvsMsg`
- It uses a helper (`caseD_12 @ 0x0249d540`) that matches **bincode-style varint markers**:
  - inline byte for values `< 0xfb`
  - extended markers `0xfb..0xfe` for larger integers
- Observed parser behavior:
  - `version == 0` → normal parse path
  - `version == 1` → special sentinel/empty path
  - other versions → `UnsupportedVersion`
- Error cluster still includes:
  - `InvalidMagic`
  - `InvalidLength`
  - `UnsupportedVersion`
- Contains RocksDB snapshot data

## 2026-04-01 Ghidra Sweep

### Encoding evidence
- `rmp` string refs are adjacent to gossip/stream clusters:
  - `0x00474276`, `0x004a267e`, `0x0075ac9d`, `0x0075ad98`, `0x0075fc25`, `0x0076d97c`
- `bincode` hits in this pass were only found in generic library regions:
  - `0x002b5693`, `0x002ba6ef`, `0x002bb593`, `0x002cb4e2`, `0x002d011d`, `0x002d561b`
- New hard evidence from runtime + parser RE:
  - 4001 bodies are frequently `lz4_flex` size-prepended blocks
  - `caseD_12 @ 0x0249d540` matches bincode-style varint integer parsing (`0xfb..0xfe`)
  - loopback 4001 frames are dominated by a `u64x8 header + concise body(tag, addr, round, ...)` family
  - this live shape fits the nearby string cluster:
    - `MsgConcise`
    - `OutMsgConcise`
    - `source`
    - `msg`
    - `prev_round`
    - `tx_hashes`
    - `next_proposer`
  - some live 4001 frames decode as inner **varint-bincode** `NetworkMessage` values after small prefixes
- Working hypothesis:
  - greeting/bootstrap/state-transfer path is mixed/layered and definitely **not** reducible to raw msgpack
  - `EvmKvsMsg` uses a versioned varint-bincode-like/custom binary parser
  - at least some live control/consensus messages are wrapped varint-bincode after an additional prefix
  - msgpack should no longer be treated as the default explanation for live 4001 bodies
  - live 4001 stream frames are wrapped in an additional `[u32 len][u8 kind]` envelope before the body
  - a fresh 181-frame localhost sweep yielded 29 inline decoded small frames that all fit the same concise parser
  - a saved 20-frame remote peer capture at `/private/tmp/hl-remotecap20.BD0UJU` now confirms the same family on real peer-facing `4001` traffic:
    - all 20 frames LZ4-decompress cleanly
    - all 20 frames fit the same `u64x8 header + concise body(tag, addr, round, mid, ...)` parser after LZ4
    - rounds advance consecutively from `1235091102` through `1235091121`
    - `tag` split is exactly even in that sample: `10x tag 27`, `10x tag 28`
    - `mid` distribution in that sample is `0:5`, `1:7`, `2:1`, `3:4`, `4:3`
    - the smaller 2-8 KB decoded frames are still present and tend to carry the richer `mid = 0` branch, but the larger 58-325 KB decoded frames are still the same outer concise family rather than a completely different codec
    - multiple address values repeat across different `tag` and `mid` combinations, including `df35aee8ef5658686142acd1e5ab5dbcdf8c51e8`, `a82fe73bbd768bc15d1ef2f6142a21ff8bd762ad`, `80f0cd23da5bf3a0101110cfd0f89c8a69a1384d`, `5ac99df645f3414876c816caa18b2d234024b487`, and `5795ab6e71ecbefa255fc4728cc34893ba992d44`
    - this means the old “large peer frames are opaque LZ4 blobs” read was too weak; what remains unresolved is the deeper meaning of the `mid = 1/2/3/4` compact branches, not whether they belong to the same outer family
  - **CONFIRMED** `mid` discriminant mapping via `objdump` of the serde Serialize function at VA `0x042b92f0`:
    - the function uses a jump table at VA `0x65f47c` with 5 relative-offset entries
    - dispatch logic: loads `[rsi]`, applies niche encoding (`cmova` with `0x8000000000000000` offset), indexes into jump table
    - each branch was traced to the exact `LEA rax, [rip+...]` instruction loading the string label:
      - Index 0 -> VA `0x042b9334` -> LEA `"in"` at VA `0x660531` (len=2)
      - Index 1 -> VA `0x042b94f2` -> LEA `"execution state"` at VA `0x660080` (len=15)
      - Index 2 -> VA `0x042b9417` -> LEA `"out"` at VA `0x660533` (len=3)
      - Index 3 -> VA `0x042b948f` -> LEA `"status"` at VA `0x65faa4` (len=6)
      - Index 4 -> VA `0x042b939f` -> LEA `"round advance"` at VA `0x660536` (len=13)
    - the confirmed wire mapping is:
      - `mid = 0` => `in` (wraps MsgConcise inner: Timeout/Tc/Heartbeat/HeartbeatAck)
      - `mid = 1` => `execution state`
      - `mid = 2` => `out` (wraps OutMsgConcise inner: ClientBlock/Qc)
      - `mid = 3` => `status`
      - `mid = 4` => `round advance`
    - **`prev_round` is a field of the outer MsgConcise struct** (alongside `source`, `msg`, `reason`), NOT a field inside the `mid` enum variant. Confirmed from field name strings at VA `0x66049c`.
    - Rust uses niche encoding: values `0x800000000000002c..=0x800000000000002f` map to indices 0-3, all other values (data pointers) default to index 4
  - across those small frames:
    - outer `tag` remained `27` or `28`
    - `mid` stayed in a small range (`0`, `1`, `2`)
    - `mid = 0` forms a stable extended branch:
      - two varint `u32` fields
      - marker byte `0x51`
      - one more varint `u32`
      - one trailing `u32` currently parses as `round - 1`, but that field name is still heuristic from live data
    - `mid = 1/2` currently look like compact branches with short scalar fields, not the richer `mid = 0` layout
    - multiple trailing 20-byte chunks remain present after the parsed round fields
    - the same 20-byte address value reappears across both `tag 27` and `tag 28`, and across multiple `mid` values, so it currently looks more like a validator/proposer identity than a pure direction field
    - within a fixed `mid`, `tag 27` and `tag 28` share the same body layout; the current evidence does not support `tag` changing the structural parse
  - working inference: `mid` is more likely a small subtype enum than a count, and `tag 27/28` are more likely outer family/wrapper discriminants than separate body layouts

### String/xref anchors
- Peer verification / rejection cluster:
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
- EVM snapshot cluster:
  - `EvmKvsMsg` → `0x0047447d`, `0x00492df4`, `0x0075ae05`, `0x00769063`, `0x0076da5b`
  - `InvalidMagic` → `0x00492e0a`, `0x0063289c`
  - `UnsupportedVersion` → `0x00492e16`, `0x004ac514`, `0x006328a8`
  - `InvalidLength` → `0x00492dfd`, `0x0063288f`
- Confirmed heartbeat / snapshot serializer anchors from exact field-name xrefs:
  - `Heartbeat` @ `0x0076047a` → `0x0438fae6` in `FUN_0438fa90`
  - `HeartbeatAck` @ `0x00760483` → `0x043903e8` in `FUN_043903c0`
  - `validator_set_snapshot` @ `0x0076055d` → `0x0438fd04` in `FUN_0438fcc0`, `0x043b9a7e` in `FUN_043b9a10`
  - `concise_lt_hashes` @ `0x00760573` → `0x0438fd46` in `FUN_0438fcc0`, `0x043b9abe` in `FUN_043b9a10`
  - `validator_to_last_msg_round` @ `0x0076059e` → `0x0438fadf`, `0x0438fb0f` in `FUN_0438fa90`, plus `0x043bab8b` in `FUN_043baac0`
  - `random_id` @ `0x007605b9` → `0x0438fad1`, `0x0438fb08` in `FUN_0438fa90`, plus `0x043babae`, `0x043bacff` in `FUN_043baac0`
- Confirmed adjacent concise / consensus-message family anchors:
  - `tx_hashes` @ `0x0076048f` → `0x043b83ea` in `FUN_043b8330`
  - `source` @ `0x0076049c` → `0x043b8595` in `FUN_043b8520`
  - `prev_round` @ `0x007604a5` → `0x043b86e5` in `FUN_043b8670`
  - `reason` @ `0x007604af` → `0x043b871e` in `FUN_043b8670`
  - `heartbeat_ack` @ `0x007604d8` → `0x043b8aa7` (containing function unresolved through current bridge)
  - `proof_len` @ `0x007604e8` → `0x043b8e88` in `FUN_043b8dc0`
  - `txs_len` @ `0x007604f1` → `0x043b8ea5` in `FUN_043b8dc0`
  - `next_proposer` @ `0x007604fa` → `0x043b906d` in `FUN_043b8f70`
  - `suspect` @ `0x00760507` → `0x043b908d` in `FUN_043b8f70`
  - `round_to_jailed_validators` @ `0x0076050e` → `0x043b922b` in `FUN_043b91a0`
- New caller edges from exact function xrefs:
  - `FUN_043b8330` is called from `FUN_043b7620` and `FUN_04593d40`
  - `FUN_043b8670` is called from `FUN_03ff8540`
  - `FUN_043b8dc0` is called from `FUN_04593650`
  - `FUN_043b8f70` is called from `FUN_0459ece0`
  - local `objdump` now also shows `FUN_043b92f0` dispatching compact-branch discriminants `0..4`
- Field-shape lift from disassembly:
  - `FUN_043b8330` reads fields in this order:
    - `proposer`
    - `round`
    - `tx_hashes`
    - `time`
    - literal `qc`
    - literal `tc`
    - `hash`
    - local `objdump` confirms those exact anchors via:
      - `proposer` at file offset `0x1e0d10`
      - `round` at `0x65fa1b`
      - `tx_hashes` at `0x66048f`
      - `time` at `0x1dfb4c`
      - `qc` / `tc` at `0x660498` / `0x66049a`
      - `hash` at `0x1dfb74`
  - `FUN_043b8670` reads:
    - `prev_round`
    - `round`
    - `reason`
  - `FUN_043b8dc0` reads:
    - `round`
    - `time`
    - `txs`
    - `proof_len`
    - `txs_len`
  - `FUN_043b8f70` reads:
    - literal inner discriminant `Qc` vs `Tc`
    - `proposer`
    - `next_proposer`
    - `suspect`
    - `last_vote_round`
    - and local `objdump` shows it literally writes `Qc` (`0x6351`) vs `Tc` (`0x6354`) before walking the rest of the payload
- Working interpretation:
  - `FUN_043b8330` is likely a `ClientBlock`-style deserializer or closely adjacent block summary type
  - `FUN_043b8670` is likely a timeout/transition reason record
  - `FUN_043b8dc0` looks like a compact block/body summary carrying tx counts
  - `FUN_043b8f70` looks like a `Qc` / `Tc`-related consensus object with proposer/jailing metadata
- New caution:
  - the live parser field currently named `prev_round` should still be treated as a heuristic candidate, not a proved semantic match
  - the stronger proved facts are the caller edges, the shared `tag 27/28` layout within a fixed `mid`, and the repeated validator/proposer-like address across both tags
- Still unresolved:
  - exact `OutMsgConcise` xrefs were not recovered cleanly through the current headless bridge
  - exact tag `27/28` discriminant-to-variant mapping still needs a deeper lift inside the `0x043b83..0x043b90..` family
  - but the literal `Qc` / `Tc` branch inside `FUN_043b8f70` makes it clear that one inner layer of the dominant concise traffic is QC/TC-adjacent, while peer `tag 27/28` likely live at a different outer wrapper layer

### CONFIRMED: RpcUpdate Enum Structure (2026-04-01)

Definitive enum layout recovered from serde Debug/Display string blob at file offset `0x660467`. Full type path confirmed: `node::consensus::rpc::RpcUpdate`.

**Tag 27 = MsgConcise** (inbound consensus messages, 8 inner variants):

Serializer at VA `0x42b7e50`, jump table at VA `0x65f418`.
```
MsgConcise {
    Tx,            // variant 0 — transaction hash
    Block,         // variant 1 — block proposal
    Vote,          // variant 2 — vote message
    Timeout,       // variant 3 — timeout vote
    Tc,            // variant 4 — timeout certificate
    BlocksAndTxs,  // variant 5 — blocks + transactions combined
    Heartbeat,     // variant 6 — validator heartbeat
    HeartbeatAck,  // variant 7 — heartbeat acknowledgement
}
```

Field mapping to functions:
- `Block` fields: proposer, round, tx_hashes, time, qc, tc, hash → `FUN_043b8330`
- `Tc` fields: prev_round, round, reason → `FUN_043b8670`
- `Heartbeat` wraps `HeartbeatSnapshot` → `FUN_0438fa90` / `FUN_0438fcc0`
- `HeartbeatAck` → `FUN_043903c0`

Outer struct fields: `{source, msg, prev_round, reason}` — `msg` is the mid-tagged inner enum.

**Tag 28 = OutMsgConcise** (outbound consensus results, 9+ inner variants):

Serializer at VA `0x42b87e0`, jump table at VA `0x65f458`.
```
OutMsgConcise {
    Block,           // variant 0 — raw block (PROVEN)
    ClientBlock,     // variant 1 — committed block for clients (PROVEN)
    VoteJailBundle,  // variant 2 — jail vote bundle (INFERRED)
    Timeout,         // variant 3 — timeout forwarded outward (PROVEN)
    Tc,              // variant 4 — TC forwarded outward (PROVEN)
    Heartbeat,       // variant 5 — heartbeat forwarded outward (PROVEN)
    NextProposer,    // variant 6 — next proposer (INFERRED)
    Tx,              // variant 7 — transaction forwarded outward (PROVEN)
    RpcRequest,      // variant 8 — RPC request (PROVEN)
}
```

Field mapping to functions:
- `ClientBlock` fields: destination, heartbeat_ack, txs, proof_len, txs_len → `FUN_043b8dc0`
- `NextProposer` fields: next_proposer, suspect, round_to_jailed_validators → `FUN_043b8f70`
  - Inner Qc/Tc discriminant: literal bytes `0x6351`="Qc" vs `0x6354`="Tc"

**Frame parser**: VA `0x4ed746e` reads tag as u8 byte, valid range 0-28 (29 total stream frame kinds).

**Mid field (consensus flow direction)**:
```
0 = in              — inbound message received
1 = execution state  — state after execution
2 = out             — outbound message sent
3 = status          — consensus status update
4 = round advance   — round transition
```

Source: FUN_043b92f0 dispatch + label strings at 0x3abac7.

**HeartbeatSnapshot structure** (nested inside MsgConcise::Heartbeat):
```
HeartbeatSnapshot {
    validator_set_snapshot: ValidatorSetSnapshot,
    evm_block_number: u64,
    concise_lt_hashes: ConciseLtHashes,
}

ValidatorSetSnapshot {
    stakes,
    jailed_validators,
}
```

Evidence level: **CONFIRMED** from binary string analysis, not inference. Variant ordering matches serde derive macro output order.

### Heartbeat serializer update
- `FUN_0438fcc0` serializes `HeartbeatSnapshot` as:
  - `validator_set_snapshot`
  - `evm_block_number`
  - `concise_lt_hashes`
- `FUN_043b9a10` deserializes the same `HeartbeatSnapshot`
- `FUN_045a0520 -> FUN_04381a00` writes `concise_lt_hashes` as:
  - `accounts_hash`
  - `contracts_hash`
  - `storage_hash`
- `FUN_043db480` / `FUN_04387640` are the serializer/deserializer pair for `ConciseLtHash { n_elements, n_bytes, hash_concise }`
- `FUN_0438fa90` / `FUN_043baac0` are a parent serde pair above `HeartbeatSnapshot`
  - confirmed parent fields: `validator`, `round`, `snapshot`, `validator_to_last_msg_round`, `random_id`, `prev_round`, `reason`
- `hash_concise` is now understood as a 32-byte SHA-256 digest on the wire, not the raw 2048-byte LtHash16 accumulator
- Adjacent type-name anchors in the same cluster:
  - `ConciseLtHashes` → `0x0075fb1a`
  - `EvmTxHash` → `0x0075fb11`
  - `SharePx` → `0x0075fb29`
  - `UsdcPx` → `0x0075fb31`
- Important nuance:
  - `FUN_04390050` also references the same names, but its caller `FUN_047c23c0` emits punctuation/comma/braces, so it is likely a text/debug/display formatter rather than the live heartbeat serializer

## GossipStatus Struct
```rust
struct GossipStatus {
    initial_height: u64,
    latest_height: u64,
}
```
Found at: `0x474b44`, `0x75b11b`

## Connection Health
States: `fresh`, `stale`, `missing`
- `"local height is from stale hardfork"` — hardfork version mismatch
- `"local height is too far behind"` — sync gap too large

### CONFIRMED: Live 4001 Send Path (2026-04-01)

Complete 6-layer caller chain from concise serializers to TCP write, traced via objdump disassembly.

**Wire order**: `bincode-fork serialize → LZ4 compress body → write [u32 BE len][u8 kind=0x01][LZ4 body]`

| Layer | VA | Role |
|-------|-----|------|
| 1 (field ser) | `0x043b8330` | tx_hashes field serializer |
| 1 (field ser) | `0x043b8670` | prev_round field serializer |
| 1 (field ser) | `0x043b8dc0` | proof_len field serializer |
| 1 (field ser) | `0x043b8f70` | next_proposer field serializer |
| 2 (struct) | `0x04593d40` | MsgConcise struct builder |
| 2 (struct) | `0x04593650` | OutMsgConcise struct copy (~0x4130 bytes) |
| 2 (struct) | `0x0459ece0` | Serialize impl for concise state |
| 3 (routing) | `0x042f8fc0` | Ring buffer producer (stream msg future) |
| 3 (routing) | `0x043a4ee0` | Consumer callback + merge-sort + wake |
| 4 (serialize) | `0x043bf670` | `send_to_destination` (route + serialize) |
| 4 (serialize) | `0x03fa33d0` | Serialize with optional LZ4 flag |
| 4 (serialize) | `0x0474f3f0` | Core bincode-fork serializer (writes tag 27/28 as u32 LE) |
| 5 (dispatch) | `0x0409c0e0` | BTreeMap peer routing (memcmp on 20-byte validator addrs) |
| 5 (dispatch) | `0x04387860` | Tokio mpsc channel sender (lock-free ring buffer) |
| 6 (TCP write) | `0x01cdae00` | `server_node_tcp_sender` task init |
| 6 (TCP write) | `0x0265c180` | `tcp_chunk` writer (LZ4 compress + framing) |
| 6 (TCP write) | `0x01cc9d00` | `tcp_bytes` state machine (`bswap` for BE len header) |

Key details:
- Tag 27/28 is written as a u32 LE variant index by the bincode-fork serializer in Layer 4
- LZ4 compression happens in Layer 6, AFTER full serialization
- `kind=0x01` means LZ4 compressed; `kind=0x00` would be raw/uncompressed
- Frame header: `[u32 BE body_len][u8 kind][LZ4-compressed bincode body]`
- Peer dispatch uses a BTreeMap keyed by 20-byte validator addresses

### CONFIRMED: Heartbeat Wire Layout (2026-04-02)

Complete byte-level layout from disassembly of serializer chain. Corrections from string blob analysis applied.

```
MsgConcise::Heartbeat (tag 27, inner variant 6):
  MsgConcise outer: {source, msg, prev_round, reason}

  Heartbeat payload (parent wrapper FUN_0438fa90 / FUN_043baac0):
    [20B]     validator (Address)
    [u64]     round
    HeartbeatSnapshot (FUN_0438fcc0 / FUN_043b9a10):
      ValidatorSetSnapshot (2 fields, NOT 3):
        stakes          -- Vec/HashMap of validator stakes
        jailed_validators -- set of jailed validator addresses
      [u64]   evm_block_number  (at rodata 0x1de290, struct offset +0xD8)
      ConciseLtHashes (FUN_04381a00):
        accounts_hash:  ConciseLtHash
        contracts_hash: ConciseLtHash
        storage_hash:   ConciseLtHash
    [HashMap] validator_to_last_msg_round  (parent field, NOT in ValidatorSetSnapshot)
    [u64]     random_id                     (parent field, NOT in ValidatorSetSnapshot)
  [u64]     prev_round  (MsgConcise outer field)
  [enum]    reason      (MsgConcise outer field)

ConciseLtHash (FUN_043db480 / FUN_04387640, 48 bytes in memory):
  hash_concise: [u64 len=32][32 bytes]  -- SHA-256 digest of full LtHash16, NOT raw 2048B
  n_elements:   u64
  n_bytes:      u64
```

**Critical: `hash_concise` is 32 bytes (SHA-256), not 2048 bytes.** The full LtHash16 (1024 × u16 = 2048B) is compressed to SHA-256 for wire transmission. Full accumulators must come from ABCI state snapshots, not heartbeats.

### CONFIRMED: Validator Transition Flow (2026-04-02)

The non-validator → consensus transition sequence:

1. `"running non-validator to get initial abci state for consensus"` (0x002d0f88)
2. `"non-validator abci state not caught up yet"` — polls until synced (0x002c839a)
3. `"using initial abci state from non-validator"` — state captured (0x002d0fb9)
4. `"client block send failed: Unbounded send error"` — **EXPECTED**, channel close is the intended termination mechanism
5. `"ending non-validator after failed client block send"` (0x002c2515)
6. `"starting bootstrap"` → `"finished bootstrap"` — consensus bootstrap phase (0x002d81b0, 0x002c4e5f)
7. `"spawning gossip rpc"` — fully operational (0x002b9264)

There is NO explicit "entered consensus" log message. Transition is implicit when bootstrap completes.

### CONFIRMED: Signer Epoch Activation (2026-04-02)

**"Signer invalid or inactive for current epoch"** check at VA `0x03355790`:
- Calls validator lookup at `0x02eb8fb0`
- Searches `epoch_states[cur_epoch]` for the 32-byte signer key via `memcmp`
- The signer must be in the current epoch's active validator set
- The active set = **top N validators by stake** (N=50 on testnet)
- If validator is below the stake threshold, signer NEVER enters the epoch state
- This is NOT a timing issue — it's a **stake ranking** issue

**Staking state fields** (from struct at 0x464820):
```
epoch_states, active_epoch, cur_epoch_state, cur_epoch,
epoch_duration_seconds, allowed_validators,
jailed_signers, jail_vote_tracker, jail_until,
signer_to_last_manual_unjail_time,
self_delegation_requirement,
disabled_validators, disabled_node_ips, validator_to_state
```

**Unjail cooldown**: "Can only manually unjail once every {} days" (0x002c37a3)

**Jailed-but-active validators** (32 on testnet) can still:
- Observe consensus messages
- Receive heartbeats
- Track rounds
- But NOT sign/vote/produce blocks

### CONFIRMED: Validator Admin Surfaces (2026-04-02)

**80 total action variants** in the main Action enum.

**CSignerAction** (variant 50, 3 sub-variants):
| Variant | Purpose |
|---------|---------|
| `unjailSelf` | Self-unjail (rate-limited by `signer_to_last_manual_unjail_time`) |
| `jailSelf` | Voluntary jail before shutdown |
| `jailSelfIsolated` | Jail in isolated mode |

**CValidatorAction** (variant 14, 16 sub-variants):
registerAsset, registerAsset2, setOracle, haltTrading, insertMarginTable, setFeeRecipient, setMarginTableIds, setOpenInterestCaps, setFundingMultipliers, setFundingInterestRates, setSubDeployers, setMarginModes, setFeeScale, setGrowthModes, disableDex, setPerpAnnotation

**Bridge/Signing Actions** (validator-only):
| Action | Purpose |
|--------|---------|
| VoteEthDeposit (37) | Vote to confirm ETH L1 deposit |
| ValidatorSignWithdrawal (33) | Sign withdrawal for bridge out |
| SignValidatorSetUpdate (23) | Sign validator set update for bridge |
| VoteEthFinalizedValidatorSetUpdate (38) | Vote on finalized validator set update |
| VoteEthFinalizedWithdrawal (39) | Vote on finalized withdrawal |

**VoteAppHash** (variant 40): `{height: u64, appHash: [u8;32], signatures}` — tracked in `app_hash_vote_tracker`, quorum at `quorum_app_hashes`. Saves `/tmp/abci_state_with_quorum_app_hash_mismatch.rmp` on mismatch.

**Override file**: `/override_consensus_rpc_c_signers.json` — local bypass for CSignerAction gating (not on-chain).

## Key Function Addresses (from Ghidra)
- `gossip_server_connect_to_peer`: `0x474220`, `0x75ac52`
- `abci_stream send tcp greeting`: `0x4a2929`
- `abci_stream recv greeting`: `0x4a2935`
- `connection_checks`: `0x4a295f`
- `verify_rpc` / `verified gossip rpc`: `0x4a2ca1` / `0x4a2cab`
- `failed to verify peer rpc`: `0x4a29ed`
- `received abci greeting from`: `0x2c8531`
- `rpc connect`: `0x2c8553`
- `compute_abci_greeting`: `0x75a122`, `0x760c97`
- `compute_no_abci_greeting`: `0x760cb8`

## Recovery Model (Durability)
On crash/restart:
1. Load latest `periodic_abci_state` snapshot (.rmp, ~1.1GB)
2. Replay `replica_cmds` blocks from snapshot height to tip
3. Reconstruct full in-memory state

Storage layout:
- **L1 state**: in-memory only, periodic msgpack snapshots to disk
- **EVM state**: RocksDB (`EvmStateDbHome`)
- **Blocks**: NDJSON files (`replica_cmds/`)
- **EVM hashes**: JSON (`lt_hashes.json` per checkpoint)

## RPC Backfill / Block Catch-Up (2026-04-01 RE)

Complete reverse engineering of how lagging peers catch up. Full details in `crates/hl-re/src/findings/rpc_backfill.rs`.

### Source Files (from binary panic paths)
```
node/src/gossip_stream.rs      — stream driver, bootstrap, forward_client_blocks
node/src/gossip_rpc.rs         — GossipRpcResponse, gossip_handle_rpc
node/src/gossip_rpc_client.rs  — outbound RPC requests
node/src/local_abci_state.rs   — staleness detection
node/src/consensus/rpc.rs      — RpcRequest, RpcUpdate, consensus rpc driver
node/src/consensus/mempool.rs  — handle_blocks_and_txs, throttling
node/src/consensus/client_block.rs — ClientBlock validation, CommitProof
```

### RpcRequest (from binary strings)
```rust
enum RpcRequestContent {
    BlockTx,
    BlocksAndTxs { after_round: u64, n: u64, last_block_hash: Hash },
}
```

### GossipRpcResponse (from type path in binary)
```rust
enum GossipRpcResponse {
    ClientBlocks(Vec<ClientBlock>),
    Peers(Vec<NodeIp>),
    GossipStatus(GossipStatus { initial_height, latest_height }),
}
```

### Staleness Detection
```
1. Query >= 2 peers for GossipStatus
2. Compare local_height vs latest_height
3. Classify: fresh | stale | missing
   - stale reasons: "local height is from stale hardfork", "local height is too far behind"
   - if stale/missing → full bootstrap (ABCI state + EVM KVs)
   - if fresh → incremental BlocksAndTxs RPC
```

### Peer Selection: Round-Robin
```
rpc_clone_main_validators → get active validator set
rpc_task_get_peers → resolve validator IPs
For each RPC request:
  1. Rpc task outbound → pick next validator
  2. send gossip rpc request → TCP to peer
  3. Peer response | Peer timed out → advance to next
  4. All exhausted → RpcRoundRobin error
```

### ClientBlock Validation (9 checks)
```
1. ClientBlockQcRound     — QC round matches block round
2. ClientBlockQcHash      — QC hash matches block hash
3. ClientBlockTx          — tx content valid
4. ClientBlockTxHashes    — tx_hashes == hash(txs)
5. ClientBlockTime        — monotonically increasing
6. ClientBlockMissingCommitProof — proof must be present
7. CommitProofChildQc     — child QC references parent
8. CommitProofGrandchildQc — grandchild QC references child
9. CommitProofConsecutive  — rounds are consecutive
```

### Bootstrap Safeguards
- `"bootstrap timeout"` — deadline on full state transfer
- `"too many blocks to request"` — range limit
- `"bootstrap ended past original block, is peer malicious?"` — overshoot detection

### Transition to Live
After backfill: `forward_client_blocks` loop, receiving `OutMsgConcise::ClientBlock` (tag 28 variant 1) via live gossip stream. `"mempool refresher"` runs in background to fill any gaps.

## Links
- [[ABCI State Machine]] — block processing pipeline
- [[HyperBFT]] — consensus protocol
- [[App Hash]] — state hashing (in heartbeats)
- [[Exchange State]] — 56-field state object
- [[LtHash Serialization]] — hash format details

#gossip #networking #protocol #confirmed
