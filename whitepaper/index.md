# White Paper

The white paper is the system-design view of the Hyperliquid reimplementation effort. It explains what the chain is trying to do, how the major subsystems fit together, and which safety and solvency properties the implementation is meant to preserve.

This repo is not an official Hyperliquid codebase. It is a reverse-engineered implementation and research environment. The right way to read this paper is:

- `what the system is for`
- `what components exist`
- `how value and control move through the system`
- `what invariants matter`

For protocol-detail and field-order truth, use the [Yellow Paper](../yellowpaper/index.md). For the fastest compact fact surface, use the [Truth Register](../findings/truth-register.md). For the live HTML reference, use the [Hypurrliquid Paper](../paper/index.html). For the actual execution and topology flows, use the dedicated [Block Lifecycle](../block-lifecycle/index.html) and [Liquidation & ADL](../liquidation/index.html) pages.

## Fact Surfaces

The narrative in this paper should be read together with the structured fact surfaces:

- [Truth Register](../findings/truth-register.md) for the compact confirmed/active/hypothesis ledger
- [Open Claims](../status/open-claims.md) for unresolved truth-maintenance work
- [Protocol Scope Matrix](../yellowpaper/protocol-scope-matrix.md) for `protocol` vs `testnet_impl` vs `local_impl`

## 1. Abstract

Hyperliquid is a purpose-built exchange chain that combines:

- perpetual futures
- spot markets
- account abstraction and portfolio margin
- lending via BOLE
- prediction markets via HIP-4 outcomes
- a tightly coupled EVM execution surface

The core engineering problem is not just matching orders. It is running a low-latency exchange state machine, a validator consensus protocol, bridge flows, and EVM-originated actions while preserving deterministic replay and app-hash parity.

This repo exists to reconstruct that system from deployed binaries, snapshots, runtime artifacts, official docs, and observed network behavior.

## 2. System Model

At a high level, the system is composed of five interacting planes:

1. Consensus
   HyperBFT orders blocks, rotates proposers, and finalizes execution.

2. Exchange state machine
   The `Exchange` object is the central L1 state surface. It owns market state, balances, books, clearinghouse logic, validator-linked trackers, delayed actions, EVM-facing state, BOLE, and more.

3. Matching and clearing
   Orders mutate books, fills mutate balances and positions, and clearing logic enforces margin and liquidation rules.

4. EVM bridge surface
   HyperEVM runs alongside the L1 state machine and re-enters L1 through CoreWriter-delayed actions and transfer queues.

5. Control and governance
   Validators, staking, bridge signatures, SetGlobal/VoteGlobal actions, and chain-level safety toggles sit above ordinary trading flow.

The system is easiest to understand as one deterministic execution engine with multiple ingress lanes.

## 3. Architectural Layers

### 3.1 Consensus and Networking

The consensus layer decides block order, proposer rotation, and commitment. The networking layer carries:

- validator-to-validator consensus traffic
- gossip and peer sync
- state/bootstrap handoff for catching-up nodes
- runtime signals like concise hashes and validator vote surfaces

The dedicated lifecycle reference is here:

- [Block Lifecycle](../block-lifecycle/index.html)

### 3.2 Exchange Execution

The exchange execution layer is the heart of the protocol. A block generally follows this shape:

1. recover / identify actors
2. run `begin_block` hook surfaces
3. deliver signed actions
4. run block-finalization logic
5. compute response hash / state hash material
6. feed commit/vote surfaces

The execution engine owns:

- order books
- balances
- perp positions
- funding trackers
- liquidation and ADL
- BOLE
- outcomes
- EVM-facing state
- validator-linked trackers and guards

### 3.3 Product Families

The protocol is not one homogeneous market engine. It contains distinct product families:

| Product family | Core behavior | Main risk surface |
| --- | --- | --- |
| Perps | matched books + clearinghouse | margin, liquidation, ADL |
| Spot | spot books and balances | transfer routing, spot-specific bookkeeping |
| Portfolio margin | unified risk over eligible assets | portfolio maintenance ratio and liquidation ordering |
| BOLE | borrow/lend pool | utilization, health factor, market/partial/backstop liquidation |
| Outcomes | 1x outcome markets | settlement and collateral conservation |
| EVM-originated actions | delayed CoreWriter path | delayed execution ordering and safety gating |

### 3.4 Universe Model

The implementation is converging on an explicit universe model:

- core perp universe: `""`
- singleton spot universe: `"spot"`
- named HIP-3 perp universes: `dexName`

That matters because transfers and balances do not all live in one flat pool. A correct node has to know which universe an asset or collateral movement belongs to before it can apply it.

## 4. Account and Risk Modes

Hyperliquid separates *where assets live* from *how risk is computed*.

The important account/risk modes are:

- classic
- dex abstraction
- unified account
- portfolio margin

These are not separate universes. They are overlays on top of the asset/universe model. That distinction is essential for building routing and liquidation correctly.

## 5. Solvency and Safety Goals

The system has several non-negotiable safety properties:

### 5.1 Deterministic State Evolution

All validators must reach the same post-block state from the same ordered input stream. That means:

- same action decode
- same execution order
- same response serialization
- same LtHash updates

### 5.2 Margin and Liquidation Safety

Risk-bearing products must not leave losses unaccounted for. The system therefore needs:

- correct maintenance checks
- deterministic liquidation triggers
- a consistent fallback path when liquidation is insufficient
- ADL where necessary

### 5.3 Collateral Conservation

Collateral should not be created by:

- bad transfer routing
- incorrect BOLE repayment accounting
- incorrect outcome settlement or merge logic
- inconsistent bridge finalization

### 5.4 Controlled EVM Re-entry

HyperEVM is not allowed to bypass L1 fairness or ordering constraints. The CoreWriter + ActionDelayer path exists to slow and serialize EVM-originated actions before they re-enter the main L1 state machine.

## 6. Liquidation and ADL

Perp and portfolio-margin liquidation are part of the main risk engine. BOLE liquidation is a separate family with its own health-factor and backstop semantics. Outcomes are not ordinary perp-style liquidations; they are primarily a settlement and collateral-conservation problem instead.

Use the dedicated execution reference for the detailed flow:

- [Liquidation & ADL](../liquidation/index.html)

## 7. EVM, Bridge, and Delayed Actions

The EVM is tightly integrated but not sovereign over L1 state. The important bridge points are:

- L1 transfers to and from HyperEVM
- CoreWriter-delayed actions
- bridge validator signatures and finalized withdrawal state

This repo now models the delayed-action queue explicitly, but the exact binary placement of matured delayed actions within the `begin_block` / execution-state wrapper is still one of the remaining ordering-closure tasks.

## 8. Outcomes and Special Risk Surfaces

The outcomes system is one of the structurally richest parts of the protocol. It introduces:

- question-level metadata
- named outcomes
- fallback outcomes
- settlement transitions
- merge/split/negation-like token mechanics

The current leading safety hypothesis in this repo is that the hardest outcome risk sits in question-level reconciliation, especially around fallback and settled named outcomes, not in the simplest binary pair mechanics alone.

## 9. Implementation Strategy

This project is not trying to write “a similar exchange.” It is trying to reach protocol truth in a controlled way.

The working method is:

1. ingest evidence
2. promote evidence into claims
3. implement only confirmed or chain-scoped truths
4. add regressions
5. run crate tests and replay/parity checks
6. sync docs and claims when repo truth moves

The docs system mirrors that:

- [Findings](../findings/index.md)
- [Status](../status/index.md)
- [Protocol Scope Matrix](../yellowpaper/protocol-scope-matrix.md)

## 10. Current Boundaries

The repo has made major progress, but some of the hardest parity surfaces are still open:

- exact response hashing parity across mainnet and testnet paths
- deeper `begin_block` per-effect semantics
- some bridge and staking transition details
- deeper outcome reconciliation logic
- ADL and liquidation edge semantics

Those open edges do not change the architectural picture. They define where the remaining engineering effort is concentrated.

## 11. Reading Guide

Use the docs in this order:

1. [Paper](../paper/index.html) for the broad RE reference
2. [White Paper](./index.md) for the system-design narrative
3. [Yellow Paper](../yellowpaper/index.md) for protocol-truth and execution details
4. [Liquidation & ADL](../liquidation/index.html) for risk execution
5. [Block Lifecycle](../block-lifecycle/index.html) for block flow and topology
6. [Findings](../findings/index.md) and [Status](../status/index.md) for active truth maintenance
