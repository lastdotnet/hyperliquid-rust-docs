# LtHash Serialization Format

Confirmed element serialization for each LtHash16 category.

## EVM State Hashes (CONFIRMED from lt_hashes.json)

Element sizes derived from `n_bytes / n_elements`:

### accounts_hash — 92 bytes per element
```
address:    20 bytes  (Address)
balance:    32 bytes  (U256 big-endian)
nonce:       8 bytes  (u64 big-endian)
code_hash:  32 bytes  (B256)
            --------
Total:      92 bytes
```
Format: Raw byte concatenation (NOT MessagePack)

### contracts_hash — ~5303 bytes per element (variable)
```
address:    20 bytes  (Address)
code:       variable  (raw bytecode bytes)
```
Format: Raw byte concatenation

### storage_hash — 84 bytes per element
```
address:    20 bytes  (Address)
slot:       32 bytes  (B256)
value:      32 bytes  (B256)
            --------
Total:      84 bytes
```
Format: Raw byte concatenation

## L1 State Hashes (from binary RE, 2026-04-01)

### ConciseLtHash struct (3 elements, named msgpack)
```
struct ConciseLtHash {
    n_elements: u64,            // count of elements hashed
    n_bytes: u64,               // total bytes processed
    hash_concise: HashMap<HashCategory, [u8; 2048]>,  // per-category LtHash16 checksums
}
```

Serialization: **Named MessagePack map** with keys in declaration order:
1. `n_elements`
2. `n_bytes`
3. `hash_concise`

Field order confirmed from binary rodata at 0x65fb37:
`n_elementsn_bytesConciseLtHashhash_concise`

The wire/gossip format transmits SHA-256 digests (32B) per category, not full
2048-byte accumulators. The ABCI state stores full accumulators.

### HashCategory enum (11 variants, confirmed order)
Rodata at 0x65fb61, verified all 11 string offsets:

| Index | Variant | Binary offset | String length |
|-------|---------|--------------|---------------|
| 0 | CValidator | 0x75fb61 | 10 |
| 1 | CSigner | 0x75fb6b | 7 |
| 2 | Na | 0x75fb72 | 2 |
| 3 | LiquidatedCross | 0x75fb74 | 15 |
| 4 | LiquidatedIsolated | 0x75fb83 | 18 |
| 5 | Settlement | 0x75fb95 | 10 |
| 6 | NetChildVaultPositions | 0x75fb9f | 22 |
| 7 | SpotDustConversion | 0x75fbb5 | 18 |
| 8 | BoleMarketLiquidation | 0x75fbc7 | 21 |
| 9 | BoleBackstopLiquidation | 0x75fbdc | 23 |
| 10 | BolePartialLiquidation | 0x75fbf3 | 22 |

Note: The category name "BoleMarketLiquidation" (not just "MarketLiquidation") and
"Na" (not "None" or "General") are the exact serde variant names from the binary.

### hash_concise category mapping (CONFIRMED from RE 2026-04-01)

**Critical finding:** The RespHash does NOT hash raw state mutations. It hashes
the **response structs** (ConsensusTxResp) produced by block execution. All 11
hash categories serialize the SAME response enum struct via `rmp_serde::to_vec`
(compact/positional format). The category only determines which LtHash16
accumulator receives the hashed element.

The `rmp` string at 0x65fc25 immediately follows the categories, confirming
rmp_serde is used for element serialization.

#### Dispatch function (vaddr 0x205a300)
- Takes `sil` = response type enum (0-19, checked with `cmp $0x13`)
- Jump table at vaddr `0x37fe2c` dispatches to 14 unique code blocks
- Each block copies a 0x800-byte LtHash16 accumulator from BSS to stack
- Constructs serializer context objects (Arc-wrapped, 0x90-byte stride descriptors)
- Serializes the response via Rust trait objects into a buffer
- Buffer goes through blake3 XOF (2048B) -> paddw u16 -> XOR into accumulator

#### Response type -> category accumulator mapping (CONFIRMED 2026-04-01)
20 response types map to 14 unique accumulators. Accumulator addresses confirmed
by extracting LEA + MOV edx,0x800 patterns from each dispatch code block:

| Accum | Address    | Dispatch entries | Hash category (confirmed)                         |
|-------|------------|------------------|----------------------------------------------------|
|  0    | 0x50066c8  | [6]              | NetChildVaultPositions                              |
|  1    | 0x5006ec8  | [4]              | LiquidatedIsolated                                  |
|  2    | 0x50076c8  | [5]              | Settlement                                          |
|  3    | 0x5007ec8  | [0, 1]           | CValidator + CSigner                                |
|  4    | 0x50086c8  | [18]             | type18 (unidentified)                               |
|  5    | 0x5008ec8  | [7, 8]           | SpotDustConversion + BoleMarketLiquidation           |
|  6    | 0x50096c8  | [19]             | type19 (unidentified)                               |
|  7    | 0x5009ec8  | [2, 3]           | Na + LiquidatedCross                                |
|  8    | 0x500a6c8  | [15]             | type15 (unidentified)                               |
|  9    | 0x500aec8  | [16]             | type16 (unidentified)                               |
| 10    | 0x500b6c8  | [11]             | type11 (unidentified)                               |
| 11    | 0x500bec8  | [17]             | type17 (unidentified)                               |
| 12    | 0x500c6c8  | [9, 10]          | BoleBackstopLiquidation + BolePartialLiquidation    |
| 13    | 0x500cec8  | [12, 13, 14]     | types 12+13+14 (shared, unidentified)               |

Note: Only 11 of 20 dispatch entries have confirmed variant names (types 0-10).
Types 11-19 likely represent additional response contexts (TWAP, trigger orders,
batch operations, etc.) but their variant names are not stored in the same string
blob as types 0-10.

#### Response field serializers (DECOMPILED 2026-04-01)

The response serialization happens through two parallel code paths at 0x427xxxx:
1. **JSON serializers** -- produce the rmp_serde byte stream for hashing
2. **Hash serializers** -- accumulate per-field LtHash values

##### JSON serializers (rmp_serde output, feeds into blake3)

**SerializerA** at VA 0x427dfb0 -- Minimal response (SpotDust-like)
```
 1. coin         Coin(String)
 2. szi          f64
```

**SerializerB** at VA 0x427e860 -- TWAP Order response
```
 1. coin         Coin(String)
 2. user         Address
 3. side         Side(enum)
 4. sz           f64
 5. executedSz   f64
 6. executedNtl  f64
 7. minutes      u64
 8. reduceOnly   bool
 9. randomize    bool
10. timestamp    u64
```

**SerializerC** at VA 0x427ead0 -- TWAP Status response
```
 1. running      Option<T>
 2. twapId       u64
 3. error        Option<T>
```

**SerializerD** at VA 0x427eeb0 -- Cancel response
**CORRECTED 2026-04-02**: Cancel response format is `{"status":"success"}` (a status
string), NOT `{"success": null}`. The cancel response is a StatusResult, not a
SimpleResult.
```
 1. status       String     // e.g. "success"
```

**SerializerE** at VA 0x427f120 -- Status-only response
```
 1. status       String
```

**SerializerI** at VA 0x427fd10 -- Order Fill response
**CORRECTED 2026-04-02**: FillResponse has **24 fields on mainnet** (not 10 or 21).
The response uses an **externally-tagged OrderStatus enum** (see App Hash doc),
NOT a flat struct. The OrderStatus variants are:
- `{"filled": {"totalSz": f64, "avgPx": f64, "oid": u64}}`
- `{"error": "string"}`
- `{"waitingForFill": {"oid": u64}}`
- `{"waitingForTrigger": {"oid": u64}}`
- `{"resting": {"oid": u64, "cloid": Cloid}}`

The full FillResponse wraps OrderStatus with additional fill context fields
including coin, user, side, sz, startPosition, hash, fee, feeToken, tid,
twapId, builderFee, deployerFee, liquidation, liquidatedUser, markPx,
triggerCondition, isTrigger, isPositionTpsl, and the OrderStatus itself.
Total: **24 fields** in the mainnet serde output.

Previous 10-field model was from a simplified older binary version.

##### Hash serializers (per-field LtHash accumulation)

**SerializerF** at VA 0x427f200 -- TWAP Status hash
```
 1. status       hash_field
 2. running      hash_field
 3. error        hash_field
 4. twapId       hash_field
```

**SerializerG** at VA 0x427f850 -- Success/Error/Status hash
```
 1. status       hash_field
 2. success      hash_field
 3. error        hash_field
```

**SerializerH** at VA 0x427f980 -- Status-only hash
```
 1. status       hash_field
```

**SerializerJ** at VA 0x4280c90 -- Order Fill hash
```
 1. filled               hash_field
 2. error                hash_field
 3. waitingForFill       hash_field
 4. waitingForTrigger    hash_field
 5. resting              hash_field
 6. totalSz              hash_field
 7. avgPx                hash_field
 8. oid                  hash_field
 9. cloid                hash_field  (via LEA rdx, different calling convention)
10. oid                  hash_field  (second: resting order)
11. cloid                hash_field  (via LEA rdx)
```

CRITICAL: The hash serializer field order DIFFERS from the JSON serializer order.
In SerializerJ, `error` comes before `waitingForFill` and `resting`, whereas in
SerializerI the order is filled->totalSz->avgPx->oid->error->waitingFor*->resting.

##### Call target type identification
| Call VA    | Purpose                      | Used for fields                    |
|------------|------------------------------|------------------------------------|
| 0x44773b0  | Coin (String) serializer     | coin                               |
| 0x446b1c0  | Address serializer           | user                               |
| 0x4490140  | Side enum serializer         | side                               |
| 0x448d640  | f64 serializer               | sz, executedSz, executedNtl, totalSz, avgPx |
| 0x4478a90  | u64 serializer               | minutes, oid, timestamp, twapId    |
| 0x4489710  | bool serializer              | reduceOnly, randomize              |
| 0x43ceae0  | Option<T> serializer         | running, filled, error, resting, etc. |
| 0x4400110  | hash_field (LtHash)          | all fields in hash serializers     |
| 0x44715e0  | String serializer            | status                             |
| 0x447a0c0  | Cloid serializer             | cloid                              |

The response field names (from binary rodata at 0x65f9cb):
`sz isz executedSz executedNtl minutes reduceOnly randomize timestamp
 running twapId error status filled totalSz avgPx oid cloid resting
 waitingForFill waitingForTrigger`

Additional field names referenced from 0x1dfb80 area:
`coin` (0x1dfb80), `user` (0x1dfc70), `side` (0x1dfd08), `success` (0x65f97c)

#### Accumulator memory layout
14 accumulators stored contiguously at 0x800-byte intervals in BSS segment:
- First accumulator: vaddr 0x050066c8 (NetChildVaultPositions)
- Last accumulator: vaddr 0x0500cec8 (types 12+13+14)
- Each: 2048 bytes = 1024 u16 values (LtHash16 checksum)
- Spacing: exactly 0x800 (2048) bytes per accumulator

#### Serializer descriptor objects
Each dispatch code block constructs serializer descriptors at 0x90 (144) byte
stride in BSS. Descriptor addresses are block-specific (e.g., 0x5002e18 for
type 11, 0x5002ed8 for types 0,1). Descriptors are Arc-wrapped and populated
at runtime, not statically readable. Initialization code found at VA 0x1fb9dfc.

#### Callers of the dispatch function
The dispatch function is called indirectly via trait object vtable (Serialize impl).
No direct CALL instructions target it. It is invoked through the serde serialization
infrastructure during block response processing.

### Block processing stages (from binary at 0x539f8a)
```
RecoverUsers -> BeginBlock -> DeliverSignedActions -> Commit -> RespHash -> Check
```
`RespHash` is the stage where response hashing occurs after block execution.

### ABCI state -> hash pipeline
```
1. Block execution produces responses (resps field of ConsensusTx)
2. Each resp maps to a HashCategory based on the action/mutation type
3. Resp is serialized via rmp_serde (compact/positional format)
4. blake3 XOF extends to 2048 bytes
5. LtHash16 accumulator updated (add for insert, sub for remove)
6. Per-category accumulators compose into ConciseLtHash
7. SHA-256 of combined accumulator -> 32-byte app_hash for VoteAppHash
```

## ABCI State Struct Field Orderings (from binary RE 2026-04-01)

All field orderings below are VERIFIED by matching concatenated field name
strings in the binary rodata against known field names. Fields listed are
in serde serialization order (= Rust struct declaration order).

### Exchange (56 elements, 59 visible serde field names)
```rust
struct Exchange {
    locus,                     // 1
    perp_dexs,                 // 2
    spot_books,                // 3
    agent_tracker,             // 4
    funding_distribute_guard,  // 5
    funding_update_guard,      // 6
    sub_account_tracker,       // 7
    allowed_liquidators,       // 8
    bridge2,                   // 9
    staking,                   // 10
    c_staking,                 // 11
    funding_err_dur_guard,     // 12
    max_order_distance_from_anchor,  // 13
    scheduled_freeze_height,   // 14
    simulate_crash_height,     // 15
    validator_power_updates,   // 16
    cancel_aggressive_orders_at_open_interest_cap_guard,  // 17
    last_hlp_cancel_time,      // 18
    post_only_until_time,      // 19
    post_only_until_height,    // 20
    spot_twap_tracker,         // 21
    user_to_display_name,      // 22
    book_empty_user_states_pruning_guard,  // 23
    user_to_scheduled_cancel,  // 24
    hyperliquidity,            // 25
    spot_disabled,             // 26
    prune_agent_idx,           // 27
    max_hlp_withdraw_fraction_per_day,  // 28
    hlp_start_of_day_account_value,     // 29
    register_token_gas_auction,         // 30
    perp_deploy_gas_auction,   // 31
    spot_pair_deploy_gas_auction,       // 32
    hyper_evm,                 // 33
    vtg,                       // 34
    app_hash_vote_tracker,     // 35
    begin_block_logic_guard,   // 36
    user_to_evm_state,         // 37
    multi_sig_tracker,         // 38
    reserve_accumulators,      // 39
    partial_liquidation_cooldown,       // 40
    evm_enabled,               // 41
    validator_l1_vote_tracker, // 42
    staking_link_tracker,      // 43
    action_delayer,            // 44
    default_hip3_limits,       // 45
    hip3_stale_mark_px_guard,  // 46
    disabled_precompiles,      // 47
    lvt,                       // 48
    hip3_no_cross,             // 49
    initial_usdc_evm_system_balance,    // 50
    validator_l1_stream_tracker,        // 51
    last_aligned_quote_token_sample_time,  // 52
    user_to_evm_to_l1_wei_remaining,       // 53
    dex_abstraction_enabled,   // 54
    hilo,                      // 55
    hnode_ip,                  // 56 (note: NOT node_ip, this is the serde name)
    delegations_disabled,      // 57
    commission_bps,            // 58
    signer,                    // 59
}
```
VERIFIED: Concatenation of all 59 field names matches binary exactly.
Note: 56 elements in serialize_struct but 59 serialize_field calls suggests
3 fields are serialized conditionally or the count was set before late additions.

### Clearinghouse (18 elements, ALL CONFIRMED 2026-04-02)
```
meta(PerpMeta/6), user_states(UserStates/6), oracle,
total_net_deposit, total_non_bridge_deposit,
perform_auto_deleveraging, adl_shortfall_remaining,
bridge2_withdraw_fee, daily_exchange_scaled_and_raw_vlms,
halted_assets, override_max_signed_distances_from_oracle,
max_withdraw_leverage, last_set_global_time, usdc_ntl_scale(f64,u64),
isolated_external, isolated_oracle, moh(MainOrHip3 enum), znfn(u64)
```
**UPDATED 2026-04-02**: All 18 fields now identified. The 3 previously unknown fields are:
- `meta` = PerpMeta (6 sub-fields)
- `user_states` = UserStates (6 sub-fields)
- `znfn` = u64 fill nonce counter
The `moh` field is confirmed as a `MainOrHip3` enum discriminant.
The `usdc_ntl_scale` field is a tuple of (f64, u64).

### Staking (6 elements) -- VERIFIED
```
epoch_states, active_epoch, cur_epoch_state, cur_epoch,
epoch_duration_seconds, allowed_validators
```

### Context (12 elements, 6 visible)
```
initial_height, height, round, next_twap_id, system_nonce, mac
```
The remaining 6 fields use short serde rename codes.

### PerpDex (4 elements) -- VERIFIED
```
books, funding_tracker, twap_tracker, perp_to_annotation
```

### FundingTracker (8 elements) -- VERIFIED
```
asset_to_premiums, override_impact_usd, clamp, default_impact_usd,
hl_only_perps, use_binance_formula, perp_to_funding_multiplier,
perp_to_funding_interest_rate
```

### CValidatorState (3 elements) -- VERIFIED
```
register_time, last_signer_change_time, delegations_blocked
```

### AppHashVoteTracker (3 elements) -- VERIFIED
```
tracker, quorum_app_hashes, highest_quorum_height
```

### ActionDelayer (9 elements) -- VERIFIED
```
delayed_actions, user_to_n_delayed_actions, n_total_delayed_actions,
max_n_delayed_actions, vty_trackers, status_guard, enabled,
delayer_mode, delay_scale
```

### AgentTracker (5 elements) -- VERIFIED
```
user_to_main_agent, user_to_extra_agents, agent_to_user,
address_to_nonce_set, user_to_pending_agent_removals
```

### SubAccountTracker (2 elements) -- VERIFIED
```
user_to_sub_accounts, sub_account_to_master
```

### PendingWithdrawals (2 elements) -- VERIFIED
```
queue, user_to_total_withdrawal
```

### Hyperliquidity (2 elements) -- VERIFIED
```
spot_to_range, ensure_order_bucket_guard
```

### MultiSigSigners (2 elements) -- VERIFIED
```
authorizedUsers, threshold
```

### StreamTracker (3 elements) -- VERIFIED
```
validator_to_value, median_value, update_median_bucket_guard
```

### ErrDurGuard (3 elements) -- VERIFIED
```
err_duration, first_err_time, last_err_time
```

### SpotClearinghouse (12 elements, 6 full + 6 short codes)
```
token_to_max_supply, token_fees_collected, native_token,
spot_dusting_opted_out_users, evm_transfer_escrows,
token_to_pending_evm_contract, p, m, r, s, o, h
```
The 6 single-char codes are serde(rename) attributes.

### Locus (14 elements, all short codes)
```
cl, ss, cl, ct, xf, tr, us, tc, hn, pd, lu, aR, bl, pq
```
(These 14 two-char serde rename codes are stored as: clssclctxftrustchnpdluaRblpqusctruacvlthcm)
Probable mapping: cl=clearinghouse, ct=context, pd=perp_dex, etc.

### TokenInfo (12 elements, all short codes)
```
szDecimals, weiDecimals, dp, du, ep, mv, rp, bo, gp, ml, bw, om, bs, uv, ip, mm
```
(Stored as: szDecimalsweiDecimalsdpduepmvrpbogpmlbwombsuvipmm)

## Wire Structures (from Ghidra decompilation 2026-04-01)

### HeartbeatSnapshot (named msgpack map)
```
{
    "validator_set_snapshot": ValidatorSetSnapshot,  // offset 0x00
    "evm_block_number": u64,                         // offset 0xd8
    "concise_lt_hashes": Option<ConciseLtHashes>     // offset 0x40
}
```

### Heartbeat (named msgpack map)
```
{
    "validator": ...,
    "round": ...,
    "snapshot": HeartbeatSnapshot,
    "validator_to_last_msg_round": ...,
    "random_id": ...
}
```

Serialization functions:
- `FUN_0459ef20` — nested struct serializer (validator_set_snapshot)
- `FUN_0459e430` — named field serializer (evm_block_number, n_elements, n_bytes)
- `FUN_045a0520` — optional/nullable field serializer (concise_lt_hashes)
- `FUN_0459e030` — binary blob serializer (hash_concise = 2048-byte LtHash)

### ConciseLtHash (named msgpack map)
```
{
    "hash_concise": bytes[32],    // offset 0x00 — SHA-256 digest of the LtHash16 checksum
    "n_elements": u64,            // offset 0x20 — count of elements hashed
    "n_bytes": u64                // offset 0x28 — total bytes processed
}
```

### ValidatorSetSnapshot
```
{
    "stakes": ...,
    "jailed_validators": ...
}
```

## Testnet Binary Dispatch (2026-04-02)

The testnet binary uses a **different architecture** than mainnet for RespHash. The compiler
inlined the entire dispatch into a single ~16KB function instead of using vtable indirection.

### compute_resps_hash (VA 0x2352ff0 testnet / 0x205a300 mainnet)
- Single monolithic function with 256-entry jump table at VA 0x3aa3f8
- Response type discriminant at struct offset 0x38
- 20 response types grouped into 6 call patterns:
  - A (direct hash): types 0, 10
  - B (serde+hash): types 1,2,7,8,11,12,15,16
  - C (complex): types 9, 14
  - D (exchange): type 13
  - E (special): type 19
  - SUB (sub-variant): types 3,4,5,6,17,18

### Testnet field string locations (CONFIRMED via /proc/PID/mem runtime probe)
```
filled=0x6cc9fd    totalSz=0x6cca03   avgPx=0x6cca0a    resting=0x6cca0f
waitingForFill=0x6cca16   waitingForTrigger=0x6cca24
coin=0x6cc960      szi=0x6cc931       executedSz=0x6cc9a3  executedNtl=0x6cc9ad
minutes=0x6cc9b8   running=0x6cc9bf   error=0x6cc9c6     success=0x6cc9cb
status=0x6cc9d2    user=0x6ccade      side=0x3f0b43      reduceOnly=0x6ccba7
randomize=0x3f022a timestamp=0x3f0233  oid=0x3f0b97       cloid=0x3f0bb3
twapId=0x3a35f3
```

## Cracked Response Struct Field Orders (2026-04-02)

CONFIRMED from testnet binary VA 0x2355860 via jump table + LEA tracing.
The `#[derive(Serialize)]` impl is shared between JSON and rmp_serde paths,
so field declaration order = serialization order for both.

**CRITICAL: The hash serializer uses DIFFERENT field order than JSON for OrderFill.**

### 11 Unique Response Struct Types

**1. OrderStatus::filled** (3 fields)
```rust
struct FilledResponse { totalSz: f64, avgPx: f64, oid: u64 }
```

**2. OrderStatus::resting** (2 fields)
```rust
struct RestingResponse { oid: u64, cloid: Option<String> }
```

**3. RestingOrder** (**15 fields**, CORRECTED 2026-04-02 -- was 12)
```rust
struct RestingOrderResponse {
    coin: String,
    side: String,
    limitPx: f64,
    sz: f64,
    timestamp: u64,
    triggerCondition: String,
    isTrigger: bool,
    triggerPx: f64,
    isPositionTpsl: bool,
    reduceOnly: bool,
    orderType: String,
    origSz: f64,
    tif: String,           // ADDED: time-in-force (e.g. "Gtc", "Alo", "Ioc")
    cloid: Option<String>, // ADDED: client order ID
    children: Vec<_>,      // ADDED: child orders (TP/SL)
}
```
**CORRECTION 2026-04-02**: RestingOrderResponse has 15 fields, not 12. The 3
additional fields (`tif`, `cloid`, `children`) were added in a binary update.
The `tif` field is critical for hash matching.

**4. TwapSliceFill** (10 fields)
```rust
struct TwapSliceFillResponse {
    coin: String,
    user: Address,
    side: String,
    sz: f64,
    executedSz: f64,
    executedNtl: f64,
    minutes: u64,
    reduceOnly: bool,
    randomize: bool,
    timestamp: u64,
}
```

**5. TwapState** (3 fields)
```rust
struct TwapStateResponse { running: Option<bool>, twapId: u64, error: Option<String> }
```

**6. Fill/Trade** (**24 fields on mainnet**, CORRECTED 2026-04-02 -- was 9)
```rust
// The FillResponse wraps an externally-tagged OrderStatus enum.
// On mainnet, the full struct includes fill context + OrderStatus.
// The 9-field version below is from the testnet binary; mainnet has
// additional fields for liquidation, builder/deployer fees, etc.
struct FillResponse {
    coin: String,
    sz: f64,
    side: String,
    startPosition: f64,
    hash: String,
    fee: f64,
    tid: u64,
    feeToken: String,
    twapId: Option<u64>,
    // Additional mainnet fields (24 total):
    // builderFee, deployerFee, liquidation, liquidatedUser, markPx,
    // triggerCondition, isTrigger, isPositionTpsl,
    // px, dir, closedPnl, oid, crossed, builder, cloid,
    // + OrderStatus enum (externally tagged)
}
```
**CRITICAL**: The OrderStatus enum uses **external tagging** (serde default),
so the JSON output is `{"filled": {"totalSz": 1.0, ...}}` NOT `{"filled": true, ...}`.
Cancel response is `{"status": "success"}` NOT `{"success": null}`.

**7. SimpleResult** (2 fields)
```rust
struct SimpleResultResponse { success: bool, error: Option<String> }
```

**8. StatusResult** (1 field)
```rust
struct StatusResultResponse { status: String }
```

**9. OrderWithTiming** (5 fields)
```rust
struct OrderWithTimingResponse {
    side: String,
    sz: f64,
    reduceOnly: bool,
    randomize: bool,
    timestamp: u64,
}
```

**10. TriggerParams** (2 fields)
```rust
struct TriggerParamsResponse { triggerPx: f64, tpsl: String }
```

**11. TriggeredFill** (2 fields)
```rust
struct TriggeredFillResponse { triggered: bool, filled: bool }
```

### New Fields Discovered (not in original RE)
- `triggerCondition` (16 chars) — string field in RestingOrder
- `isTrigger` (9 chars) — boolean in RestingOrder
- `isPositionTpsl` (14 chars) — boolean in RestingOrder
- `startPosition` (13 chars) — f64 in Fill/Trade
- `feeToken` (8 chars) — string in Fill/Trade
- `triggered` (9 chars) — boolean in TriggeredFill

### Hash Path vs JSON Path Field Order Difference

For Order Fill, the hash serializer (J) uses a DIFFERENT field order than JSON (I):

**Hash path**: `filled, error, waitingForFill, waitingForTrigger, resting, totalSz, avgPx, oid, cloid`
**JSON path**: `filled, totalSz, avgPx, oid, error, waitingForFill, waitingForTrigger, resting, cloid`

This means the `#[derive(Serialize)]` for the hash path must declare fields in the hash order,
NOT the JSON order. Separate structs needed for hash serialization.

`validator_to_last_msg_round` and `random_id` are now traced on the outer
`Heartbeat` wrapper, not on `ValidatorSetSnapshot`.

### Key Offsets (in HeartbeatSnapshot struct)
- 0x00: validator_set_snapshot
- 0x40: concise_lt_hashes (Option<ConciseLtHashes>)
- 0xd8: evm_block_number

## Testnet Binary Dispatch Analysis (2026-04-02)

Binary: `/tmp/target_bin_analysis` (hl-node testnet, swapped from mainnet)
BuildID: 167a5d3f1b4b726d184d2e56e1db5ad2cfc4e1ef

### Key Functions (ELF VAs)
```
compute_resps_hash:       0x2352ff0   (inline serialization, ~16KB function)
run_node_compute_resps_hash string:  0x3ab8ff
Tracing span wrapper:     0x2362362
blake3 hash_update:       0x1d746b0   (called per element)
serde serialize helper:   0x22eb850
blake3 finalize:          0x4d08120
complex nested serialize: 0x2324440
vec serialize:            0x233d6d0
```

### Architecture Difference from Mainnet

The testnet binary uses **inline serialization** within `compute_resps_hash`,
NOT vtable-based dispatch. There are no vtable descriptors with serializer
function pointers at slot[10]. Instead, the compiler inlined the entire
dispatch chain into a single monolithic function using nested switch tables.

This is architecturally different from the mainnet binary pattern (vtable at
0x205a300). The compiler (likely a newer Rust/LLVM version) has inlined
all serializer dispatch logic.

### Primary Dispatch Table

256-entry jump table at VA `0x3aa3f8`, indexed by byte at struct offset `0x38`
(the response type enum discriminant, values 0-19).

| Type | Code VA    | Pattern                                        |
|------|-----------|------------------------------------------------|
|  0   | 0x23558bb | direct hash_update                              |
|  1   | 0x2355d62 | serde_helper + hash_update                      |
|  2   | 0x2355d55 | serde_helper + hash_update                      |
|  3   | 0x235597b | secondary dispatch (byte 0x47 -> table 0x3aa430)|
|  4   | 0x2355913 | secondary dispatch (byte 0xd70 -> table 0x3aa420)|
|  5   | 0x235593a | secondary dispatch (byte 0xd8 -> table 0x3aa410) |
|  6   | 0x2355940 | secondary dispatch (byte 0xd8 -> table 0x3aa410) |
|  7   | 0x2355da0 | serde_helper + hash_update                      |
|  8   | 0x2355d7c | serde_helper + hash_update                      |
|  9   | 0x235934d | complex_ser + vec_ser + hash_update              |
| 10   | 0x2355909 | direct hash_update                               |
| 11   | 0x2355d7e | serde_helper + hash_update                      |
| 12   | 0x2355d5a | serde_helper + hash_update                      |
| 13   | 0x2358b52 | gossip_rpc + complex_ser + vec_ser + map_ser     |
| 14   | 0x235596f | complex_ser + vec_ser + hash_update              |
| 15   | 0x2355da4 | serde_helper + hash_update                      |
| 16   | 0x2355d92 | serde_helper + hash_update                      |
| 17   | 0x2355986 | secondary dispatch (byte 0x47 -> table 0x3aa430) |
| 18   | 0x23559d1 | secondary dispatch (byte 0xf68 -> table 0x3aa4a4)|
| 19   | 0x23559e9 | special finalize path                            |

### Call Pattern Groups

| Group | Types                      | Call Signature                    |
|-------|----------------------------|-----------------------------------|
| A     | 0, 10                      | hash_update only                  |
| B     | 1,2,7,8,11,12,15,16       | serde_helper + hash_update        |
| C     | 9, 14                      | complex_ser + vec_ser + hash_update|
| D     | 13                         | gossip_rpc + complex + vec + map  |
| E     | 19                         | special finalize (0x4d081f0)      |
| SUB   | 3,4,5,6,17,18             | secondary dispatch (sub-variant)  |

### Secondary Dispatch Tables (14 unique, 256 entries each)

Types with sub-variants use a second-level jump table indexed by a different
byte from the response struct:

| Table VA  | Struct Offset | Used By Types      |
|-----------|--------------|-------------------|
| 0x3aa3f8  | 0x38         | primary (all 20)  |
| 0x3aa410  | 0xd8         | 5, 6              |
| 0x3aa420  | 0xd70        | 4                 |
| 0x3aa430  | 0x47         | 3, 17             |
| 0x3aa444  | 0x782        | (nested)          |
| 0x3aa454  | 0x77a        | (nested)          |
| 0x3aa464  | 0x770        | (nested)          |
| 0x3aa474  | 0x768        | (nested)          |
| 0x3aa484  | 0x760        | (nested)          |
| 0x3aa494  | 0x759        | (nested)          |
| 0x3aa4a4  | 0xf68        | 18                |
| 0x3aa4b4  | 0xf62        | (nested)          |
| 0x3aa4c4  | (varies)     | (nested)          |
| 0x3aa4e0  | 0x48         | (nested, 2 uses)  |

### Confirmed String Locations (Testnet Binary)

| String                        | File Offset | Context                          |
|-------------------------------|-------------|----------------------------------|
| CValidatorCSigner             | 0x3ba8bf    | HashCategory enum names          |
| run_node_compute_resps_hash   | 0x3ab8ff    | Tracing span name                |
| filled                        | 0x6cc9fd    | Response field (order fill)      |
| totalSz                       | 0x6cca03    | Response field (order fill)      |
| avgPx                         | 0x6cca0a    | Response field (order fill)      |
| waitingForFill                | 0x6cca16    | Response field (order fill)      |
| resting                       | 0x6cca0f    | Response field (order fill)      |
| waitingForTrigger             | 0x6cca25    | Response field (order fill)      |
| running                       | 0x6cc9bf    | Response field (TWAP status)     |
| error                         | 0x6cc9c6    | Response field (generic)         |
| success                       | 0x6cc9cb    | Response field (generic)         |
| status                        | 0x6cc9d2    | Response field (generic)         |
| delayedUntil                  | 0x6cc9d8    | Response field (delayed action)  |
| twapId                        | 0x3a35f3    | TWAP identifier field            |

### JSON API Serializer Functions (separate from hash path)

These functions serialize responses for the /info API, not for hashing:

| VA        | Type              | Fields                                              |
|-----------|-------------------|-----------------------------------------------------|
| 0x4a75120 | Order fill        | filled, totalSz, avgPx, resting, waitingFor*, oid, cloid |
| 0x4a74910 | Order fields      | oid, crossed, fee, builderFee, tid, cloid, liquidation |
| 0x4a70000 | TWAP status       | executedSz, executedNtl, minutes, running, error, status |
| 0x4a72000 | Delayed action    | delayedUntil, height                                |
| 0x4a7e000 | TWAP state        | activated, terminated, twap_id, executed_sz, slice_number |
| 0x4a6c000 | Exchange state    | szi, pxs, coin_to_mark_px, coin_to_oracle_px       |
| 0x4b69000 | Running status    | running, error, success, status                     |
| 0x4bdd000 | Success/error     | success, error                                      |
| 0x4af6000 | Status only       | status                                              |
| 0x4a76000 | Display name      | display_name                                        |

### Call Chain: run_node_compute_resps_hash

```
tracing_wrapper (0x2362362)
  -> LEA 'run_node_compute_resps_hash' (span name)
  -> CALL compute_resps_hash_a (0x2352ff0) or compute_resps_hash_b (0x2353300)
       -> switch on byte[0x38] (primary dispatch table 0x3aa3f8)
         -> for types 0,1,2,7,8,10,11,12,15,16: inline serialize + CALL hash_update
         -> for types 3,4,5,6,17,18: secondary switch on sub-variant byte
         -> for types 9,14: CALL complex_serialize + vec_serialize + hash_update
         -> for type 13: CALL gossip_rpc_serialize + complex chain
         -> for type 19: CALL special_finalize
```

## Links
- [[App Hash]] — overall hash structure
- [[LtHash]] — algorithm details
- [[Exchange State]] — state being hashed
- [[ABCI State Machine]] — block processing

#hashing #serialization #confirmed
