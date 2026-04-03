# Status

This section is the current operating dashboard for the repo.

Use it when the question is:

- what is implemented?
- what is still open?
- where is drift likely?
- what should be worked next?

## 1. Current Operating State

As of the current 2026-04-03 docs pass:

- `hl-engine` crate tests were green earlier in the session
- the docs system now has:
  - a docs hub
  - a paper page
  - dedicated liquidation and block-lifecycle pages
  - white paper and yellow paper surfaces
  - a structured claims/sources layer
  - a public docs export path

The key point is that the repo is no longer missing a documentation model. The remaining work is mostly about keeping the implementation, claims, and rendered docs in sync.

## 2. Best Status Surfaces

### 2.1 Active Coding / RE Lanes

- [Task List](../../TASK_LIST.md)

This is the main live work queue and handoff note surface.

### 2.2 Open Protocol / Implementation Questions

- [Open Claims](./open-claims.md)
- [Truth Register](../findings/truth-register.md)

This is the best short list of unresolved truth surfaces.

### 2.3 Drift Detection

- [Protocol Sync Report](./protocol-sync-report.md)

This is the current docs/code/claim consistency check.

### 2.4 Maintenance Workflow

- [Doc Sync Workflow](./doc-sync-workflow.md)

This is the process that keeps RE claims, docs, and code from drifting apart.

## 3. Current Open Buckets

### 3.1 Execution Ordering

Still-open execution questions include:

- deeper begin-block semantics inside the now-closed 9-effect ordering
- ActionDelayer mode/control behavior
- per-effect guard cadence and threshold closure

### 3.2 Hashing and Replay

Still-open parity questions include:

- full response serialization closure across chains
- remaining replay mismatch localization
- complete response-hash routing coverage

### 3.3 Risk and Liquidation

Still-open risk questions include:

- ADL details
- remaining liquidation edge semantics
- full portfolio-margin and BOLE interactions

### 3.4 Outcomes

Still-open outcome questions include:

- `MergeQuestion` / fallback reconciliation
- settlement/merge gating
- deeper collateral-conservation proof

## 4. Current Strengths

The repo now has good structure in these areas:

- explicit universe routing model
- explicit product-family split for liquidation
- explicit block-lifecycle page and hook inventory
- explicit claims/source promotion model
- real public docs publishing path

These are not the final protocol answers, but they are strong foundations for ongoing implementation.

## 5. Recommended Working Loop

The best working loop for this repo is:

1. pick an open claim or active task
2. inspect the strongest subsystem note
3. implement the narrowest confirmed fix
4. add a regression
5. run crate tests
6. update claims/docs if repo truth moved
7. rerun the sync report if the change is structural

## 6. Reading Order

For engineering status:

1. [Task List](../../TASK_LIST.md)
2. [Open Claims](./open-claims.md)
3. [Protocol Sync Report](./protocol-sync-report.md)

For documentation status:

1. [Docs Hub](../index.html)
2. [White Paper](../whitepaper/index.html)
3. [Yellow Paper](../yellowpaper/index.html)
4. [Findings](../findings/index.md)

For protocol implementation status:

1. [Block Lifecycle](../block-lifecycle/index.html)
2. [Liquidation & ADL](../liquidation/index.html)
3. [Action Inventory](../yellowpaper/action-inventory.html)
