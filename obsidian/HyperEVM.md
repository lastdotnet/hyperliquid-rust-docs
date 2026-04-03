# HyperEVM

The EVM layer integrated into [[Hyperliquid]]. Runs inside the L1 node using revm.

## Specs
- **Hardfork**: Cancun (no blobs)
- **Chain ID**: 999 (mainnet), 998 (testnet)
- **Fast blocks**: 1 second, 3M gas (updated from 2M)
- **Large blocks**: 60 seconds, 30M gas
- **Fees**: Base + priority fees both burned

## Dual-Block Architecture
Two interleaved block types sharing one block number sequence:
- Fast blocks: regular transactions (1s/3M gas)
- Large blocks: complex deploys (60s/30M gas)
- Users route to large blocks via `evmUserModify.usingBigBlocks`

## Read Precompiles (0x0800+)
EVM contracts can read L1 state:
- Perp positions, spot balances
- Vault equity, staking delegations
- Oracle prices, L1 block number
- Gas cost: `2000 + 65 * (input_len + output_len)`
- `highest_precompile_address` — dynamic registration
- `disabled_precompiles` — selective disabling

## CoreWriter (0x3333...3333)
EVM contracts can write to L1:
- 15 action types (orders, transfers, delegation, etc.)
- Burns ~25,000 gas
- Emits logs processed by L1 as action intents
- **Actions are delayed** by [[Action Delayer]] (prevents latency advantage)
- Failed actions do NOT revert the EVM transaction

## System Transactions
- L1 injects `SystemTx` into EVM blocks
- Handles L1→EVM asset transfers (token minting)
- Custom RPC: `eth_getSystemTxsByBlockHash/Number`

## Token System
- HYPE: native gas token at `0x2222...2222`
- Other tokens: `0x20` prefix + token index big-endian
- Core→EVM: `sendAsset` to system address
- EVM→Core: ERC20 transfer to system address

## State (HyperEvm with 13 fields)
```
state2, latest_block2
small_block_context, big_block_context
is_initialized
l1_to_evm_native_token_queue
l1_to_evm_erc20_queue
l1_transfers_enabled
disabled_core_writer_actions
...
```

## Our Implementation
`hl-evm/` crate:
- revm 36 with Cancun spec
- L1StateProvider trait for precompile data
- EngineStateProvider backed by our matching engine
- JSON-RPC server for Foundry/Anvil compatibility
- Deploy, execute, fund operations

## sendToEvmWithData (11 fields, CONFIRMED 2026-04-02)

The `sendToEvmWithData` action type sends assets from HyperCore L1 to HyperEVM with optional calldata. **11 fields confirmed**:

```rust
struct SendToEvmWithData {
    token: u32,                        // token index
    amount: String,                    // amount as string (decimal)
    sourceDex: u32,                    // source DEX index (0 = main)
    destinationRecipient: Address,     // EVM recipient address
    toPerp: bool,                      // route to perp (HIP-3)
    destinationChainId: u64,           // EVM chain ID (999 mainnet)
    time: u64,                         // timestamp
    data: Vec<u8>,                     // calldata for EVM execution
    nonce: u64,                        // replay protection
    signatureChainId: String,          // chain ID for signature
    hyperliquidChain: String,          // "Mainnet" / "Testnet"
}
```

## Links
- [[Exchange State]] — `hyper_evm` is field 33
- [[Action Types]] — `evmRawTx`, `sendToEvmWithData`
- [[Guards and Limits]] — `disabled_core_writer_actions`, `disabled_precompiles`
- [[Binary Structure]] — revm 36 (confirmed in Cargo.toml)

#evm #ethereum #precompiles
