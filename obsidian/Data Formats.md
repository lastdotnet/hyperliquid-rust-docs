# Data Formats

The various data formats used by [[Hyperliquid]] for state, blocks, and communication.

## Replica Commands (NDJSON)
Path: `replica_cmds/{start_time}/{date}/{height}`
Format: Newline-delimited JSON

```json
{
  "abci_block": {
    "time": "2026-03-31T10:26:14.624305951",
    "round": 1233482130,
    "parent_round": 1233482129,
    "hardfork": {"version": 81, "round": 1221967841},
    "proposer": "0x80f0cd23...",
    "signed_action_bundles": [
      ["0xhash", {
        "signed_actions": [{
          "signature": {"r": "0x...", "s": "0x...", "v": 27},
          "action": {"type": "order", "orders": [...]},
          "nonce": 1774952773999,
          "vaultAddress": "0x..." // optional
        }],
        "broadcaster": "0x940e4f78...",
        "broadcaster_nonce": 1774952552281
      }]
    ]
  }
}
```

## ABCI State Snapshots (.rmp)
Path: `periodic_abci_states/{date}/{height}.rmp`
Format: MessagePack with **named keys** (NOT compact mode)
Size: ~1.1GB per snapshot, saved every ~20 minutes

Structure: `{"exchange": {56 named fields}}`
Can be translated to JSON: `hl-node --chain Mainnet translate-abci-state <in.rmp> <out.json>`
JSON output: ~5.5GB

## Node Fills (NDJSON)
Path: `node_fills_by_block/hourly/{date}/{hour}`
Format: NDJSON, ~250MB per hour

```json
{
  "block_number": 941006631,
  "block_time": "2026-03-31T08:59:59.852527297",
  "events": [
    ["0xuser_address", {
      "coin": "xyz:MSTR",
      "px": "123.63", "sz": "1.5", "side": "B",
      "closedPnl": "0.3135", "fee": "-0.001854",
      "oid": 366158135200, "tid": 1086003134703173,
      "cloid": "0x19d40a3b...",
      "startPosition": "-80.25", "dir": "Close Short"
    }]
  ]
}
```

## Gossip Wire Format
Port 4001 (streaming): `[u32 BE = 2] [00 00 00]` → `[u32 BE = 1] [00] [chain]`
Port 4002 (RPC): `[u32 BE = 1] [u8] [u8 role]`
Serialization: MessagePack (compact/positional mode for wire, named for storage)

## LtHash Checksums
Path: `evm_db_hub_slow/checkpoint/{height}/lt_hashes.json`
Format: JSON with byte arrays

```json
{
  "accounts_hash": {
    "n_elements": 1344484,
    "n_bytes": 123692528,
    "hash": [211, 88, 140, ...] // 2048 bytes = LtHash16
  }
}
```

## Book Orders (in state)
Found in `exchange.perp_dexs[].books[].book_orders`:
```json
{
  "0": {
    "p": 41025,       // price (raw, /10 = actual)
    "r": 150,         // remaining size (raw, /1e5 for szDec=5)
    "O": 150,         // original size
    "u": "0xaddr",    // user address
    "t": "Gtc",       // time-in-force
    "o": 351297023194, // order ID
    "c": {
      "t": 1773700516723, // timestamp
      "S": 150,           // original size
      "r": true,          // reduce_only
      "s": "A",           // side (A=ask, B=bid)
      "l": 813330         // level key
    }
  }
}
```

## User State (in clearinghouse)
```json
{
  "u": 10269381274,    // USDC balance (/1e6 = dollars)
  "p": {
    "p": [
      [0, {             // asset 0 = BTC
        "l": {"C": 15}, // cross leverage 15x
        "M": 56,        // margin table ID
        "f": {"a": -217821} // accumulated funding
      }]
    ]
  }
}
```

## Links
- [[Gossip Protocol]] — wire format details
- [[Exchange State]] — state structure
- [[Matching Engine]] — order format
- [[App Hash]] — lt_hashes.json format

#formats #data #serialization
