# Implementation Roadmap

What's needed to complete hlx as a consensus-compatible Hyperliquid node reimplementation.

Updated 2026-04-02 (end of session).

---

## Session Summary (2026-04-02)

**Binary RE**: 16 agents completed across every major system. ~200 confirmed claims with VA addresses.

**Implementation**: 2-chain HyperBFT (21 tests), mempool service (7-stage admission), evidence-labeled code.

**Validator**: HypurrLiquid testnet validator running on hyperscan (64.34.94.231), unjailed, in active set (#42).

**Documentation**: Purple paper (8 chapters + evidence map), 9 new obsidian pages, complete implementation roadmap.

---

## What's Done

### RE Complete + Implemented + Tests Passing

| System | Tests | Evidence Level |
|--------|-------|----------------|
| HyperBFT 2-chain consensus | 21 | CONFIRMED (VA 0x6600fa, 0x76db94) |
| Mempool service | 10 | CONFIRMED (error strings, struct fields) |
| Wire protocol types (MsgConcise/OutMsgConcise/ConciseMid) | 3 | CONFIRMED (jump tables, serializer VAs) |
| Gossip connection + block streaming | 1 | CONFIRMED + OBSERVED (20-frame capture) |
| Matching engine (basic) | 12+ | CONFIRMED (35 order statuses) |
| Clearinghouse (basic) | — | CONFIRMED (18-field layout) |
| LtHash16 + state hash framework | — | CONFIRMED (algorithm cracked) |
| EVM executor (revm 36) | — | Implemented |
| Devnet + mock services | 17 | Implemented |

### RE Complete, Needs Implementation

| System | Key Findings | Priority |
|--------|-------------|----------|
| Block building pipeline | 8-phase RecoverUsers→Commit, 76 broadcasters, no priority auction | HIGH |
| Liquidation engine | ADL by PnL rank, no socialized losses, margin tiers | HIGH |
| Fee schedule | 6 VIP + 3 MM tiers, staking/referral/builder discounts | HIGH |
| Matching engine (full) | TWAP, triggers, OCO, self-trade prevention | HIGH |
| Funding rate | 1h settlement, rate/8h, fraction-based EMA (num/denom) | HIGH |
| Oracle / mark price | Feeds → validator votes → median → mark → impact | HIGH |
| RPC backfill | BlocksAndTxs range sync, 9 validation checks | HIGH |
| HLP vault model | 13-field Vault, parent/child, withdrawal limits | MEDIUM |
| HIP-4 outcomes | 8 deploy + 4 user actions, settlement | MEDIUM |
| EVM precompiles | ReadPrecompile gas=2000+65*(in+out), CoreWriter intents | MEDIUM |
| Bridge mechanics | 4-validator committee, deposit/withdrawal lifecycle | MEDIUM |
| Governance | GovPropose/GovVote, delegation-weighted | LOW |

### RE Recently Closed, Needs Wiring

| System | Status |
|--------|--------|
| Response/hash field ordering | CRACKED: 11 response structs with exact serialization order, not yet wired into `StateHasher` |

---

## What's Unknown

### Critical (blocks app-hash parity)

| Unknown | Evidence Level | Best Attack |
|---------|---------------|-------------|
| L1 accumulator starting value / replay bootstrap point | UNKNOWN | Replay from trusted snapshot or recover from validator/runtime evidence |
| Exact premium → funding formula math | INFERRED | Instrument live funding vs oracle/mark spread |
| Cross-broadcaster ordering within a block | UNKNOWN | Analyze consecutive blocks for tx patterns |
| begin_block edge-case semantics (guard cadence, stale-mark/prune criteria) | CONFIRMED skeleton | **9 effects confirmed** (VA 0x01e748e0); guard cadence/thresholds still need work |
| Margin calculation formula (initial vs maintenance) | INFERRED | Compare against /info API margin responses |
| Block hash algorithm | UNKNOWN | Could be Blake3, SHA-256, or other |

### Important (blocks full validator parity)

| Unknown | Evidence Level |
|---------|---------------|
| Epoch transition trigger (when does ForceIncreaseEpoch fire?) | UNKNOWN |
| Empty block timing (EmptyBlockTimer interval) | UNKNOWN |
| Heartbeat send frequency | UNKNOWN |
| Latency EMA jail threshold value | UNKNOWN |
| Leader election TTL algorithm details | INFERRED (RoundRobinTtl string confirmed, algorithm not) |
| `use_binance_formula` behavior (how does it differ?) | UNKNOWN |
| Bridge validator selection criteria | UNKNOWN |

### Acknowledged Gaps in Implementation

| Gap | Notes |
|-----|-------|
| `block_store` is Vec not HashMap | O(n) lookup, should be O(1) |
| `block_hash_to_block` in mempool unbounded | No pruning/eviction |
| 3x `parse_address` duplication | Should consolidate to hl-primitives |
| Dead types in funding.rs | Oracle, FeeComputer, OrderStatus unused |
| Stringly-typed action dispatch | 90 variants matched on raw strings |
| `state_hash.rs` still hashes placeholder tuples for fills/settlements | RE field orders are cracked, but live hashing path is not fully upgraded yet |
| `DeterministicEma` 5th field | **CONFIRMED** — `last_sample_time` as ISO 8601 String |
| `DeterministicVty` window size 1000 | INFERRED — no binary evidence |
| `DEFAULT_STALE_TIMEOUT = 60s` | INFERRED — not from binary |
| `MAX_COMMITTED = 100_000` | INFERRED — not from binary |

---

## Critical Path to App-Hash Parity

```
1. Replace placeholder tuple hashing with recovered response structs
   │
2. begin_block effects (9 effects, exact ordering CONFIRMED)
   │  distribute_funding → update_funding_rates → cancel_aggressive_at_oi_cap →
   │  prune_book_empty_user_states → validator_l1_vote_prune → reset_recent_ois →
   │  refresh_hip3_stale_mark_pxs → update_stale_mark_guards → apply_bole_liquidations
   │
3. Action dispatch (90 variants, each must match)
   │  Start with: Order, Cancel, UsdSend, TokenDelegate
   │
4. end_block effects + hash computation
   │
5. Funding formula (EMA math)
   │
6. Oracle/mark price pipeline
   │
7. Fee computation + distribution
   │
8. Margin/liquidation cascade
   │
9. EVM block building (small/big)
   │
10. Replay validation
    Run follower against replica_cmds
    Compare app_hash at checkpoints
    Debug divergences
```

Steps 1-4 are serial dependencies. 5-9 can be parallelized.

---

## Next Session Priorities

### Next Session Priorities
1. Wire the recovered response struct field ordering into `StateHasher`
2. Use `hlx replay-validate` against replica_cmds to localize first divergence
3. Build remaining begin_block/end_block effects with evidence labels
4. Implement full fee computation (tiers, discounts, distribution)
5. Implement funding rate settlement with fraction-based EMA
6. Implement oracle price aggregation pipeline

---

## Evidence Confidence Summary

| Level | Count | Standard |
|-------|-------|----------|
| CONFIRMED | ~200+ | Binary string/disassembly with VA address or file offset |
| OBSERVED | ~30 | Live traffic validation (20-frame capture, API responses) |
| INFERRED | ~40 | Pattern matching, reasonable assumption, no direct proof |
| UNKNOWN | ~15 | Explicitly acknowledged — we don't know and say so |

All implementation choices trace to [[HyperBFT Protocol Specification]] Appendix C.

---

## Links
- [[HyperBFT Protocol Specification]] — purple paper (single source of truth)
- [[Gossip Protocol]] — wire format
- [[HIP-4 Outcomes]] — prediction markets
- [[Liquidation Engine]] — ADL + margin
- [[Matching Engine]] — order book mechanics
- [[Oracle and Mark Price]] — price pipeline
- [[Funding]] — funding rate formula
- [[Project Status]] — current build status

#roadmap #implementation #planning
