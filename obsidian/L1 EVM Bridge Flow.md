# L1-EVM Bridge Flow

Complete data flow for cross-layer asset transfers between HyperCore (L1) and HyperEVM.

## Direction 1: L1 → EVM (`sendToEvmWithData`)

### Step 1: User submits signed action

```
User → Broadcaster → L1 mempool → block inclusion
```

The action is a `HyperliquidTransaction` with these fields (from binary rodata):

```rust
HyperliquidTransaction {
    payload: Action,                    // the sendToEvmWithData action
    multi_sig_user: Option<Address>,    // multi-sig wrapper if applicable
    outer_signer: Option<Address>,      // for agent-signed txs
    wei: u64,                           // gas fee in wei
    nonce: u64,                         // replay protection
    destination: Address,               // broadcaster address
    source_dex: u32,                    // source DEX index (0 = main)
    destination_dex: u32,               // destination DEX index
    token: u32,                         // token being transferred
    amount: String,                     // amount as decimal string
    system_address: Option<Address>,    // system action source
    signature_chain_id: String,         // EIP-712 chain ID
    // ... more fields for other action types
}
```

### Step 2: L1 validates and debits

File: `l1/src/exchange/exchange.rs` (binary), our `exchange.rs`

```
SendToEvmWithData handler:
  1. Parse amount from string → f64
  2. Check user has sufficient balance (try_debit)
  3. If insufficient → reject with "insufficient balance or invalid amount"
  4. If sufficient → debit from clearinghouse balance
  5. Record hash status: success
```

**No whitelist check** — any user can call `sendToEvmWithData`. The protection is purely economic (you must have the balance).

### Step 3: Enqueue transfer

The L1 doesn't execute the EVM side immediately. Instead it enqueues the transfer:

```rust
// struct SendToEvmData (5 fields from binary serde)
SendToEvmData {
    destination_recipient: Address,  // EVM destination
    destination_chain_id: u64,       // 999 (mainnet) or 998 (testnet)
    gas_limit: u64,                  // gas for EVM execution
    // + 2 more fields (data? token?)
}
```

The transfer goes into one of two queues on `HyperEvm`:
- `l1_to_evm_native_token_queue` — for HYPE native transfers
- `l1_to_evm_erc20_queue` — for ERC20 token transfers

### Step 4: System transaction injection

In the next EVM block, the L1 node drains the queues and creates **system transactions**:

```
begin_block (EVM):
  for each entry in l1_to_evm_native_token_queue:
    Create SystemTx that mints native HYPE to destination
  for each entry in l1_to_evm_erc20_queue:
    Create SystemTx that mints ERC20 to destination
```

The system transactions are special — they bypass gas accounting and are injected by the node, not by any user. They appear in blocks via `eth_getSystemTxsByBlockNumber`.

### Step 5: EVM execution

If the `sendToEvmWithData` included `data` (calldata):
- The system tx calls the `destination_recipient` contract with the calldata
- This enables **atomic L1→EVM transfers + contract calls** (e.g., deposit to a vault contract)
- If the EVM call reverts, **the transfer still succeeds** — the tokens are minted but the calldata execution fails silently

## Direction 2: EVM → L1 (CoreWriter)

### Step 1: EVM contract calls CoreWriter

```solidity
// CoreWriter at 0x3333...3333
interface ICoreWriter {
    event RawAction(address indexed user, bytes data);
    function sendRawAction(bytes calldata data) external;
}
```

The `data` bytes are a **JSON-encoded L1 action** — the same format as the REST API. For example, to place an order:

```solidity
ICoreWriter(0x3333...3333).sendRawAction(
    '{"type":"order","orders":[...],"grouping":"na"}'
);
```

### Step 2: CoreWriter emits log

- Burns ~25,000 gas
- Emits `RawAction(msg.sender, data)` log event
- The EVM tx succeeds regardless of whether the L1 action will succeed

### Step 3: L1 processes logs

After the EVM block executes, the L1 node scans all logs from CoreWriter:

```
end_evm_block:
  for each log from CORE_WRITER_ADDRESS:
    if topic[0] == RawAction.selector:
      user = topic[1] (indexed address)
      data = log.data (bytes)
      action = json_parse(data) as HlAction
      
      // Check if this action type is enabled
      if action_type_index in disabled_core_writer_actions:
        skip
      
      // Queue for delayed execution (NOT immediate!)
      action_delayer.enqueue(user, action)
```

### Step 4: Action delayer

CoreWriter actions go through the **Action Delayer** (`action_delayer` on Exchange, 9 fields):

```rust
ActionDelayer {
    delayed_actions: Vec<DelayedAction>,
    user_to_n_delayed_actions: HashMap<Address, u64>,
    n_total_delayed_actions: u64,
    max_n_delayed_actions: u64,
    vty_trackers: ...,
    status_guard: BucketGuard,
    enabled: bool,
    delayer_mode: ...,  // e.g., Alternating
    delay_scale: f64,   // multiplier for delay duration
}
```

Actions are delayed by several seconds before execution. This prevents EVM contracts from getting a latency advantage over L1-native traders.

### Step 5: Delayed execution

In a subsequent `begin_block`, the delayer checks for matured actions:

```
begin_block:
  for each delayed_action where time >= matured:
    execute as if it were a normal L1 action
    remove from queue
```

**Failed actions are silently dropped** — they don't revert anything on the EVM side.

## Direction 3: EVM → L1 (ERC20 transfer to system address)

A simpler path for token transfers without action dispatch:

```
EVM user → ERC20.transfer(SYSTEM_ADDRESS, amount) → L1 credits balance
```

The system address intercepts ERC20 transfers and credits the sender's L1 balance. This is tracked in `user_to_evm_to_l1_wei_remaining`.

## Security Model

### What stops abuse?

1. **No ERC20 approve attack vector** — CoreWriter doesn't call `transferFrom`. It only reads `msg.sender` from the log event. An EVM contract can only submit actions *for itself*, not for other users.

2. **Disabled action types** — `disabled_core_writer_actions` is a list of action type indices that are blocked. Validators can disable any action type via `SetCoreWriterActionEnabled` governance vote.

3. **Action delayer** — All CoreWriter actions are delayed (seconds, not immediate). This prevents front-running and MEV extraction by EVM contracts.

4. **Gas cost** — 25,000 gas per CoreWriter call limits spam.

5. **Balance validation** — L1 still validates all actions normally. Insufficient balance, invalid parameters, etc. all cause the action to fail (silently).

6. **Frozen users** — `frozen_users` list (from `FreezeUser` spot deploy action). Frozen users can't transfer. Special `DeployerSendToEvmForFrozenUserAction` exists for deployers to recover frozen user funds.

7. **`evm_transfer_escrows`** — Escrow tracking on SpotClearinghouse prevents double-spending during cross-layer transfers.

8. **`l1_transfers_enabled`** — Global kill switch on HyperEvm. If false, all cross-layer transfers are blocked.

### Known attack surfaces

1. **Calldata execution** — `sendToEvmWithData` with calldata executes arbitrary EVM calls with the minted tokens. A malicious contract could re-enter or do unexpected things. But since the transfer is already committed on L1 before EVM execution, this is limited to EVM-side effects.

2. **Log parsing** — The L1 trusts CoreWriter logs. If the CoreWriter contract were compromised (impossible since it's a precompile, not upgradeable), fake actions could be injected.

3. **Nonce gaps** — `address_to_nonce_set` on AgentTracker tracks nonces. But CoreWriter actions don't have L1 nonces (they have EVM tx nonces). The action delayer provides ordering instead.

## Key Structs (from binary serde chain)

| Struct | Fields | Purpose |
|--------|--------|---------|
| `HyperEvm` | 13 | L1-side EVM state, queues, config |
| `SendToEvmData` | 5 | Per-transfer metadata |
| `L1ToEvmNativeTokenTransfer` | 2 | Native HYPE queue entry |
| `L1ToEvmErc20Transfer` | 5 | ERC20 queue entry |
| `ActionDelayer` | 9 | CoreWriter action delay queue |
| `EvmState` | 2+ | Per-user EVM state |
| `EvmArchiveBlock` | 5 | Archived EVM block data |

## Source Modules (from binary)

```
l1/src/evm/hyper_evm.rs     — HyperEvm state management
l1/src/evm/transactor.rs    — EVM tx execution, system tx injection
l1/src/evm/evm_state.rs     — Per-user EVM state
l1/src/evm/evm_db.rs        — EVM state database
l1/src/evm/hash.rs          — EVM state hashing (accounts, contracts, storage)
evm_rpc/src/evm_write_forwarder.rs — CoreWriter log forwarding
evm_rpc/src/evm_rpc.rs      — JSON-RPC server
```

## Links
- [[HyperEVM]] — EVM specs, precompiles, block architecture
- [[Action Delayer]] — delay queue for CoreWriter actions
- [[Exchange State]] — `hyper_evm` field [32], `action_delayer` field [43]
- [[Action Types]] — `sendToEvmWithData`, `evmRawTx`

#evm #bridge #security #cross-layer
