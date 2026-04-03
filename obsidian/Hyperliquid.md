# Hyperliquid

The Hyperliquid L1 is a purpose-built blockchain for perpetual futures, spot trading, and an integrated EVM. It uses [[HyperBFT]] consensus with ~70ms block times and supports ~200K operations/second.

## Architecture

```
┌─────────────────────────────────┐
│          HyperBFT               │  ← [[HyperBFT]]
│   24 validators, 70ms blocks    │
└────────────┬────────────────────┘
             │
┌────────────▼────────────────────┐
│          Exchange(56)            │  ← [[Exchange State]]
│  The god struct — ALL L1 state   │
├──────────┬───────────┬──────────┤
│ [[Locus]]│[[Staking]] │[[HyperEVM]]│
│ (14)     │ (6+23)    │ (13)     │
└──────────┴───────────┴──────────┘
     │
┌────▼─────────────────────────────┐
│       [[Matching Engine]]         │
│  Order books, fills, price-time   │
│  priority, sig figs (3,4,5)       │
└────┬───────────────────────┬─────┘
     │                       │
┌────▼────┐          ┌──────▼──────┐
│[[Clearing│          │[[Funding]]   │
│ house]]  │          │ DeterEma(5) │
│ (18)     │          │ DeterVty(7) │
└──────────┘          └─────────────┘
```

## Key Components
- [[Exchange State]] — 56-field god struct
- [[HyperBFT]] — consensus protocol
- [[Matching Engine]] — order books and fills
- [[Clearinghouse]] — positions, margin, liquidations
- [[HyperEVM]] — EVM with precompiles
- [[Gossip Protocol]] — peer-to-peer networking
- [[LtHash]] — state hashing
- [[Action Types]] — 80 variants
- [[Guards and Limits]] — rate limiting and protection
- [[Bridge]] — ETH deposit/withdrawal voting

## Binary
- Written in Rust, closed source
- 81MB stripped ELF binary
- Build: `105cb1dc` (2026-03-21)
- Hardfork v81 (current mainnet)

## Tags
#hyperliquid #l1 #defi #perps
