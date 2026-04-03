# Devnet

A local development network with mock services replacing the real Hyperliquid infrastructure.

## Architecture

```
                    ┌─────────────┐
    HTTP API ──────>│ Broadcaster │──────┐
  (port 3001)       └─────────────┘      │
                                         ▼
  ┌──────────┐    ┌────────────────┐   ┌───────────────┐
  │  Oracle   │──>│ Block Producer │──>│   Exchange     │
  └──────────┘    └────────────────┘   │  (L1 Engine)   │
  ┌──────────┐          │              │  57 fields     │
  │  Bridge   │─────────┘              └───────────────┘
  └──────────┘                               │
                                        ExEx events
```

## Components

### Mock Oracle (`oracle.rs`)
Generates `SetGlobalAction` blocks with oracle price updates.

**Modes:**
- **Static** — fixed prices, deterministic testing
- **Synthetic** — Brownian motion random walk (LCG-based), funding/liquidation testing
- **Live** — pull real prices from `api.hyperliquid.xyz`

Default assets: BTC, ETH, SOL, DOGE, AVAX, MATIC, LINK, ARB, OP, SUI, HYPE

### Mock Bridge (`bridge.rs`)
Simulates the Arbitrum ↔ Hyperliquid bridge.

- **Deposits**: Submit via `/devnet/deposit` → auto-confirmed after N blocks → credited to user balance
- **Withdrawals**: `withdraw3` action → auto-signed after N blocks → released
- Configurable delays (default: 1 block for dev convenience)
- Tracks total deposited/withdrawn

### Mock Broadcaster (`broadcaster.rs`)
Receives user-signed actions and bundles them for block inclusion.

- FIFO queue with max 256 actions per bundle
- Deterministic bundle hashes from broadcaster nonce
- Compatible with existing Hyperliquid SDK action format
- Thread-safe `Arc<Mutex>` wrapper for HTTP handler

### Block Producer (`block_producer.rs`)
Produces blocks at configurable intervals (default 70ms).

Per-block pipeline:
1. Tick oracle → inject `SetGlobalAction` if due
2. Tick bridge → credit deposits, sign withdrawals
3. Drain broadcaster → collect user action bundles
4. Assemble block JSON (matches `replica_cmds` format)
5. Feed through `Exchange::process_block()`

### HTTP API (`api.rs`)
Endpoints matching `api.hyperliquid.xyz`:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/exchange` | POST | Submit signed actions (SDK-compatible) |
| `/devnet/deposit` | POST | Credit USDC to an address |
| `/devnet/status` | GET | Health check + stats |
| `/info` | POST | Query exchange state (30+ types: allMids, l2Book, meta, clearinghouseState, openOrders, userFills, orderStatus, fundingHistory, portfolio, etc.) |

## Usage

```bash
# Start devnet with defaults (10 dev accounts, $1M each, synthetic oracle)
hl-node devnet

# Static oracle, 100ms blocks, port 4000
hl-node devnet --oracle-mode static --block-interval-ms 100 --api-port 4000

# No dev accounts (manual deposit)
hl-node devnet --dev-accounts 0
```

### Submitting Actions

```bash
# Deposit USDC to an address
curl -X POST http://localhost:3001/devnet/deposit \
  -d '{"user":"0x1234...","amount":100000}'

# Submit an order
curl -X POST http://localhost:3001/exchange \
  -d '{"user":"0x1234...","action":{"type":"order","orders":[{"a":0,"b":true,"p":"85000","s":"0.1","r":false,"t":{"limit":{"tif":"Gtc"}}}]}}'

# Check status
curl http://localhost:3001/devnet/status
```

## Genesis Accounts

When `--dev-accounts N` is set (default 10), N accounts are pre-funded with $1M USDC.
Addresses are deterministic: `keccak256([i; 32])[12..32]` for i in 1..=N.

## Links
- [[Exchange State]] — the engine being driven
- [[Matching Engine]] — order matching
- [[Funding]] — funding settlement in `end_block`
- [[Bridge]] — real bridge protocol being mocked
- [[Project Status]] — overall project progress

#devnet #testing #infrastructure
