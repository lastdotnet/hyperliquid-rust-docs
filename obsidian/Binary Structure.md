# Binary Structure

The hl-node binary internals discovered via reverse engineering.

## Binary Info

### Mainnet (latest analyzed)
- **Size**: 81MB, x86_64 ELF, stripped, PIE
- **.text**: 62MB (offset 0x11d1fc0)
- **.rodata**: 6.7MB
- **Build**: `105cb1dc` (2026-03-21)
- **Dynamic libs**: libssl3, libcrypto3, libstdc++6, libc6

### Testnet (2026-04-03, latest)
- **Size**: 87MB, x86_64 ELF, stripped, PIE
- **SHA256**: `1a99c892ac1f9657c8502ce04daedee27eb47edfa076531549b963ddf2ad6a37`
- **Build**: `331bef9b` (2026-04-03 10:43:39 UTC, uncommitted=false)
- **Source root**: `/home/ubuntu/hl/code_Testnet/`
- **Key difference**: Testnet binary is 6MB larger, contains new action variants, `code_Testnet` paths
- **Commands**: `run-non-validator`, `translate-abci-state`, `send-signed-action`, `check-reachability`, `compute-referrer-states`, `compute-l4-snapshots`

## Internal Crate Structure (updated from testnet 2026-04-03)
```
/home/ubuntu/hl/code_{Mainnet,Testnet}/
├── base/src/               — Shared utilities (50+ files)
│   ├── address.rs, wallet.rs, short_string.rs
│   ├── bucket_guard.rs, c_guard.rs, err_dur_guard.rs
│   ├── ema_tracker.rs, latency_sampler.rs
│   ├── production.rs       — contains FirewallIps, handle_stream_connection
│   ├── ren.rs              — visor/restart manager
│   ├── s3.rs, http_client.rs
│   └── log/, time.rs, duration.rs
├── db/src/                 — RocksDB backend
│   ├── rocks_db.rs, block_reader.rs
│   ├── db_hub.rs, db_home.rs, dh.rs, owned_db_home.rs
├── evm_rpc/src/            — EVM JSON-RPC
│   ├── evm_rpc.rs, evm_store.rs
│   ├── evm_write_forwarder.rs, method_router.rs
│   └── reth_eth_tx_builder_copied.rs, reth_rpc_helpers_copied.rs
├── global_constants/src/   — aws_name, static_config
├── info_server/src/        — /info REST API
│   ├── abci_state_info_handler.rs, handle_request.rs
│   ├── handler.rs, router.rs, request.rs, user_ip.rs
│   └── f_types/position.rs
├── l1/src/                 — Core L1 logic (THE BIG ONE)
│   ├── abci/{engine.rs, state.rs}
│   ├── action/{c_deposit, evm_raw_tx, rmp_signable, vault_transfer, vote_global, withdraw3}
│   ├── agent_tracker.rs, sub_account.rs, system_user.rs
│   ├── asset/{asset_wire.rs, pdi.rs}
│   ├── bole/reserve.rs
│   ├── book/{book_orders, book/{mod,impl_insert_order}}, books.rs
│   ├── bridge_request.rs, bridge2.rs
│   ├── clearinghouse/{cl_trait, clearinghouse, impl_first_dex, margin_table, oracle/oracle, position, position_map, user_state, user_states}
│   ├── dex_registry/{clearinghouses, dex_registry, perp_dex}
│   ├── evm/{evm_db, evm_state, hash, hyper_evm, transactor}
│   ├── exchange/{begin_block, end_block, exchange, impl_bole, impl_global, impl_ledger, impl_outcome, impl_spot_deploy, impl_trading, impl_vault, stake_requirement}
│   ├── fees/compute.rs
│   ├── hyperliquidity.rs, action_delayer.rs
│   ├── order_wire.rs, perp_meta.rs, spot_meta.rs
│   ├── qtys/{mod, impl_px, impl_sz, impl_ntl, impl_wei, units}
│   ├── raw_signature_tracker.rs, raw_vlm.rs, scaled_vlm.rs
│   ├── spot_clearinghouse/{cl_trait, impl_outcome, spot_clearinghouse, types}
│   ├── staking.rs, time.rs, utils.rs
│   └── vault/mod.rs
├── net_utils/src/          — TCP + LZ4 networking
│   ├── abci_latency_sampler.rs, async_sleep_retry.rs
│   ├── channel.rs, node_port.rs, rate_limiter.rs, rw_lock.rs
│   ├── tcp/{lz4_stats, read, reconnecting_tcp_client, tcp_listener, tcp_stream, traffic_recorder, write}
│   └── visor.rs
└── node/src/               — Orchestration + Consensus
    ├── hl_node.rs, node.rs, startup.rs
    ├── abci_checkpoint.rs, abci_stream.rs, aux.rs
    ├── channel.rs, evm_kvs.rs, firewall_ips.rs
    ├── gossip_config.rs, gossip_rpc.rs, gossip_rpc_client.rs
    ├── group_and_seq_sampler.rs, height_boundary_liner.rs
    ├── local_abci_state.rs, nv_stream.rs, stream_msg.rs
    └── consensus/{client_block, config, execution_state, heartbeat_tracker,
                   latency_samplers, liner, mempool, network, node_ips,
                   rpc, server, signed, state, timer, types, validator_set}
```

## Forked Dependencies
- `hyper-fork/` — Custom hyper HTTP
- `hyper-tls-fork/` — Custom TLS
- `hyper-alloy-core-fork/` — Custom alloy core
- `bincode-fork/` — Custom bincode (internal serialization)

## Crate Versions (from path strings)
| Crate | Version |
|-------|---------|
| alloy-rpc-types-eth | 0.9.2 |
| alloy-consensus | 0.9.2 |
| blake3 | 1.7.0 |
| sha2 | 0.10.8 |
| tiny-keccak | 2.0.2 |
| rand_chacha | 0.3.1 |
| tokio | 1.44.2 |
| chrono | 0.4.38 |
| serde_json | 1.0.120 |
| rmp-serde | 1.3.0 |
| clap | 2.34.0 |
| revm | 19.2.0 |
| revm-precompile | 16.0.0 |

## RE Tools Used
- `hl-re` (our goblin-based tool) — string extraction, struct mapping
- `radare2` — installed on hyperscan for targeted analysis
- `Ghidra` — headless analysis (4.1GB database, 4hr timeout)
- Python — manual binary parsing, LEA instruction scanning

## Serde Struct Chain Discovery (2026-04-03)

The Rust serde `derive(Serialize, Deserialize)` macro generates **continuous `.rodata` blobs** containing `struct Name with N elements` headers followed by field names. In the testnet binary, three identical copies exist (at `0x389950`, `0x391fba`, `0x543428`). The third copy at `0x543428` includes `LtHashes` and is the most complete.

This struct chain encodes the **entire ABCI state serialization layout** — every struct, its field count, and field names in exact serialization order. This is the single most valuable RE artifact from the binary.

Full parsed chain: `docs/generated/re/runs/2026-04-03-struct-chain.md`

## Discovered Structs (200+)
See [[Exchange State]], [[Clearinghouse]], [[Matching Engine]], [[Action Types]]

## Crate Versions (from testnet binary path strings, 2026-04-03)
| Crate | Version |
|-------|---------|
| alloy-consensus | 0.9.2 |
| revm | 19.2.0 |
| revm-interpreter | 15.0.0 |
| crossbeam-epoch | 0.9.18 |
| rustc-hash | 2.1.0 |

*(Mainnet versions in original table above still apply)*

## Links
- [[Hyperliquid]] — overview
- [[Exchange State]] — 57-field god struct
- [[LtHash]] — crypto crates used
- [[Gossip Protocol]] — wire format details

#reverse-engineering #binary #architecture
