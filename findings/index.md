# Findings

This section is the curated entrypoint for reverse-engineering results, protocol contradictions, and subsystem-specific research outputs.

Use it when the question is:

- what did we learn?
- which findings are already promoted into claims?
- which research lanes are still active?
- where are the best supporting notes?

## 1. Current High-Value Findings

Start with the compact fact ledger if you need the shortest current truth surface:

- [Truth Register](./truth-register.md)

### 1.1 Exchange State and Serialization

- `Exchange` is currently tracked here as a 57-field state surface.
- the repo now has a structured sync check to keep that claim aligned across docs and code
- response hashing is chain-scoped and not yet fully closed across all mainnet/testnet serializer paths

Start with:

- [Claims Index](./claims-index.md)
- [Protocol Sync Report](../status/protocol-sync-report.md)
- [Exchange State](../obsidian/Exchange%20State.md)
- [LtHash Serialization](../obsidian/LtHash%20Serialization.md)

### 1.2 Block Ordering and Hook Surfaces

- the repo has one explicit block-lifecycle map now
- the 9 named begin-block effects are now tracked as a closed claim in-repo
- the open work on this lane is now deeper per-effect semantics and ActionDelayer mode behavior, not BOLE/action-delayer slot placement

Start with:

- [Block Lifecycle](../block-lifecycle/index.html)
- [ABCI State Machine](../obsidian/ABCI%20State%20Machine.md)
- [Truth Register](./truth-register.md)

### 1.3 Liquidation, ADL, and BOLE

- liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes
- BOLE is modeled as its own liquidation family, not a perp-style liquidation surface
- ADL remains one of the deeper remaining protocol lanes

Start with:

- [Liquidation & ADL](../liquidation/index.html)
- [Liquidation Engine](../obsidian/Liquidation%20Engine.md)
- [Portfolio margin liquidation claim](../status/open-claims.md)

### 1.4 Outcomes and Solvency

- the leading current risk hypothesis is still question-level outcome reconciliation
- fallback/named-outcome settlement interaction remains the most interesting open outcome lane
- this is a research-heavy lane, not a closed protocol chapter yet

Start with:

- [Outcome solvency review 2026-04-03](../generated/outcome_solvency_review_2026-04-03.md)
- [HIP-4 Outcomes](../obsidian/HIP-4%20Outcomes.md)
- [Outcomes](../obsidian/Outcomes.md)

## 2. Structured Findings Surfaces

### 2.1 Claims

The normalized claim inventory is here:

- [Claims Index](./claims-index.md)
- [Truth Register](./truth-register.md)

That is the best place to see which findings have already been promoted into durable repo truth.

### 2.2 Open Claims

The unresolved or still-active claim surface is here:

- [Open Claims](../status/open-claims.md)

Use that page to separate:

- active protocol questions
- chain-scoped implementation differences
- local implementation gaps

### 2.3 Scope Matrix

The current scope split is compiled here:

- [Protocol Scope Matrix](../yellowpaper/protocol-scope-matrix.md)

That is the quickest view of `protocol` vs `testnet_impl` vs `local_impl`.

## 3. Best Subsystem Notes

If you want the deeper writeups rather than the curated summaries, these are the strongest subsystem notes to start from:

- [Action Types](../obsidian/Action%20Types.md)
- [Action Delayer](../obsidian/Action%20Delayer.md)
- [Bridge](../obsidian/Bridge.md)
- [Funding](../obsidian/Funding.md)
- [Gossip Protocol](../obsidian/Gossip%20Protocol.md)
- [HyperBFT Protocol Specification](../obsidian/HyperBFT%20Protocol%20Specification.md)
- [Liquidation Engine](../obsidian/Liquidation%20Engine.md)
- [Matching Engine](../obsidian/Matching%20Engine.md)
- [Oracle and Mark Price](../obsidian/Oracle%20and%20Mark%20Price.md)

## 4. How To Use This Section

For implementation work:

1. start with [Open Claims](../status/open-claims.md)
2. find the owning subsystem note
3. cross-check the [Yellow Paper](../yellowpaper/index.md)
4. patch code only once the claim is strong enough

For review work:

1. read [Claims Index](./claims-index.md)
2. read the subsystem note
3. compare local implementation comments against the claim scope/confidence

For replay/parity work:

1. use [Block Lifecycle](../block-lifecycle/index.html)
2. use [Liquidation & ADL](../liquidation/index.html)
3. use the relevant hashing and execution claims from the index
