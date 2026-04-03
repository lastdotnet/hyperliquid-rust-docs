# Locus

The core position tracking container in [[Hyperliquid]]. Binary says "struct Locus with 14 elements", but current snapshots have 15 fields (aux added later).

**CONFIRMED 2026-04-02**: All 15 field names, types, and sub-keys extracted from ABCI state snapshot 942770000.rmp and cross-referenced against binary rodata.

## Fields (3-char abbreviated serde names)

| Idx | Key | Full Name | Type | Detail |
|-----|-----|-----------|------|--------|
| 0 | `cls` | clearinghouses | Vec\<Clearinghouse\>(9) | {meta, user_states, oracle, total_net_deposit, total_non_bridge_deposit, ...} -- Clearinghouse with 18 elements |
| 1 | `scl` | spot_clearinghouse | SpotClearinghouse(12) | Has its own "Exchange" struct internally; 1.2M user_states |
| 2 | `ctx` | context | Context(12) | Block execution context |
| 3 | `ftr` | fee_tracker | FeeTracker(10) | Fee schedules, referrals, builder fees |
| 4 | `ust` | user_states | HashMap\<Address, UserState\> | 1.8M users on mainnet |
| 5 | `chn` | chain | String | "Mainnet" / "Testnet" / "Sandbox" / "Local" |
| 6 | `pdl` | perps_disabled_launch | bool | false on mainnet |
| 7 | `uaR` | user_action_registry | Vec<?> | Empty on mainnet (capital R is notable) |
| 8 | `blp` | bole_pool | BolePool(19) | BOLE lending pool state |
| 9 | `qus` | queue_signers | Vec\<Address\> | 1 address on mainnet |
| 10 | `ctr` | counters | Counters(7) | Transaction/validation counters |
| 11 | `uac` | user_account_configs | HashMap\<Address, T\> | 30,833 entries on mainnet |
| 12 | `vlt` | vaults | HashMap\<Address, Vault\> | 9,376 vaults (Vault with 13 fields) |
| 13 | `hcm` | hot_cold_mode | String | "enabled" on mainnet |
| 14 | `aux` | auxiliary | Vec<?> | Empty on mainnet (not in binary's 14-field struct descriptor) |

## Context (ctx, 12 fields)
```
initial_height: u64      -- snapshot start height (e.g. 930593000)
hardfork: struct          -- hardfork configuration
height: u64               -- current block height (e.g. 942770000)
tx_index: u64             -- transaction index within block (e.g. 776)
round: u64                -- consensus round (e.g. 1235282531)
time: DateTime<Utc>       -- block timestamp
next_oid: u64             -- next order ID (e.g. 368028013642)
next_lid: u64             -- next liquidation ID (e.g. 26738)
next_twap_id: u64         -- next TWAP order ID (e.g. 1711548)
system_nonce: u64         -- system nonce (e.g. 940536)
use_system_nonce: bool    -- false on mainnet
mac: u64                  -- message authentication counter (0)
```

## Clearinghouses (cls)
Array of 9 [[Clearinghouse]] instances (was 8, now 9):
- `cls[0]`: Main perp clearinghouse (struct Clearinghouse with 18 elements)
- `cls[1-8]`: Additional clearinghouses for HIP-3 deployed assets
- Each contains: meta, user_states, oracle, margin tables

## BolePool (blp, 19 fields)
BOLE lending pool with abbreviated field names:
```
dp, du, o, pmmt, e, pmv, r, p, n, b, bog, pmdl, pml, uobc, ubgl, ubul, bwo, mbs, u
```
- dp/du: deposit paused/unpaused flags (bool)
- o: open (bool=true)
- pmmt: portfolio margin max tiers (int=10)
- e: enabled (bool=true)
- pmv: portfolio margin value (int=500000000)
- mbs: min borrow size (int=1000000000000)
- u: users list (7405 entries)
- Various guards and limits (bog, pmdl, pml, uobc, ubgl, ubul)

## Counters (ctr, 7 fields)
```
tn: int=895791497    -- total nonces?
vn: int=815067822    -- validated nonces?
tc: int=552534955    -- total cancels?
sc: int=234487787    -- successful cancels?
to: int=1000160565   -- total orders?
so: int=239557085    -- successful orders?
ac: dict(6)          -- action counters
```

## FeeTracker (ftr, 10 fields)
```
fee_schedule, unrewarded_users, user_states(1.3M), referrer_states(57K),
code_to_referrer(57K), referral_bucket_millis(3600000),
total_fees_collected(1124300512426352), total_spot_fees_collected(32770682260982),
collected_builder_fees(838), trials_disabled(false)
```

## SpotClearinghouse (scl, 12 fields)
Internal struct confusingly also named "Exchange" in binary. Fields:
```
meta(13), user_states(1.2M), daily_exchange_scaled_and_raw_vlms(21),
token_to_genesis(297), token_to_max_supply(282), token_fees_collected(270),
native_token(150), spot_dusting_opted_out_users(3756),
evm_transfer_escrows(906), token_to_pending_evm_contract(0), pmr(1), soh(3)
```

## Links
- [[Exchange State]] -- locus is field 0 of the 57-field Exchange
- [[Clearinghouse]] -- cls contains clearinghouse instances
- [[Matching Engine]] -- order books referenced from locus

#state #architecture #core
