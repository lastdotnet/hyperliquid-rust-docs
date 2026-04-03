# LtHash

Lattice-based Homomorphic Hash used by [[Hyperliquid]] for incremental state hashing. The `app_hash` in [[HyperBFT]] is computed using LtHash.

## Algorithm
From Meta/Facebook: "Securing Update Propagation with Homomorphic Hashing" (2019)

### Properties
- **Homomorphic**: H(A ∪ B) = H(A) + H(B) (component-wise mod 2^16)
- **Incremental**: O(1) per state mutation instead of O(n) rehash
- **Order-independent**: insert order doesn't matter

### LtHash16
- Checksum: `[u16; 1024]` = 2048 bytes
- XOF: **BLAKE3** (confirmed from [[Binary Structure]] — blake3 1.7.0 with SSE2/AVX2/AVX512)
- Final compression: likely SHA-256 (sha2 0.10.8 also present)

### Operations
```rust
// Insert element
xof_output = BLAKE3_XOF(element)  // 2048 bytes
for i in 0..1024:
    checksum[i] = checksum[i].wrapping_add(xof_output_as_u16[i])

// Remove element
for i in 0..1024:
    checksum[i] = checksum[i].wrapping_sub(xof_output_as_u16[i])

// Finalize to 32 bytes
app_hash = SHA256(checksum)  // compress 2048 → 32 bytes
```

## Hash Operations (from binary)
Each state mutation type has its own hash update:
- `CValidator` — validator state change
- `CSigner` — signer change
- `LiquidatedCross` — cross-margin liquidation
- `LiquidatedIsolated` — isolated-margin liquidation
- `Settlement` — funding settlement
- `NetChildVaultPositions` — vault position netting
- `SpotDustConversion` — dust conversion
- `Bole` — [[BOLE]] lending state
- `MarketLiquidation` — market liquidation
- `BoleBackstopLiquidation` — backstop liquidation
- `BolePartialLiquidation` — partial liquidation

## EVM State Hashes
Separate from the L1 LtHash:
- `accounts_hash` — EVM account state
- `contracts_hash` — EVM contract state
- `storage_hash` — EVM storage state

## App Hash Voting
- Validators compute `app_hash` after each block
- Vote via `VoteAppHash` action
- Quorum (2/3+ stake) → `quorum_app_hash`
- Mismatch dumps to `/tmp/abci_state_with_quorum_app_hash_mismatch.rmp`

## Implementation
Our implementation in `hl-engine/src/lthash.rs`:
- BLAKE3 XOF for element hashing
- `iter().zip()` for wrapping_add/wrapping_sub
- Stack-allocated 2048-byte buffer for finalize
- All 6 tests passing

## Remaining Unknowns
- Exact serialization format for some L1 state entries being hashed
- Which state mutations trigger which hash operations (11 L1 categories identified, exact field ordering still TBD for some)

**Resolved**: Finalization hash is **SHA-256** (CONFIRMED). Not Keccak. Algorithm fully cracked: rmp_serde -> blake3 XOF (2048B) -> paddw u16 -> SHA-256 (32B).

## Links
- [[HyperBFT]] — consensus uses app_hash
- [[Exchange State]] — state being hashed
- [[Binary Structure]] — crypto crates: blake3, sha2, tiny-keccak

#cryptography #hashing #state
