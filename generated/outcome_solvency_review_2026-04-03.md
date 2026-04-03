# Outcome Solvency Review

Date: 2026-04-03

Status: in progress

This note captures only the parts that are grounded by live testnet data, local
docs, and direct binary anchors.

## Confirmed

1. Outcomes are not margin-liquidated instruments.

- Local protocol notes describe HIP-4 outcomes as `1x isolated only`, with
  `no leverage, no liquidations`.
- Settlement behavior is `outcomeSettledCanceled` / `outcomeSettledRejected`,
  not a liquidation path.
- BOLE liquidation is a separate system with explicit categories:
  `BoleMarketLiquidation`, `BoleBackstopLiquidation`,
  `BolePartialLiquidation`.
- `SpotDustConversion` is also separate from BOLE liquidation.

Implication:
- Any outcome solvency bug is much more likely to be collateral conservation or
  question-state accounting than liquidation math.

## Live Testnet Outcome State

Official endpoint:

```bash
curl -sS 'https://api.hyperliquid-testnet.xyz/info' \
  -H 'content-type: application/json' \
  --data '{"type":"outcomeMeta"}'
```

Confirmed live question:

```json
{
  "question": 1,
  "name": "What will Hypurr eat the most of in Feb 2026?",
  "fallbackOutcome": 13,
  "namedOutcomes": [10, 11, 12],
  "settledNamedOutcomes": []
}
```

Implication:
- Question state is explicitly split into:
  - active named outcomes
  - a fallback outcome
  - settled named outcomes

That is the main solvency surface.

Official docs nuance:

- the official `outcomeMeta` docs currently describe this as a testnet-only
  spot info endpoint and show only an `outcomes` array in the response example
- the live testnet response is richer and also returns `questions`

Implication:
- the public docs are not yet a complete source of truth for question-level
  state; the live testnet payload is the better current source

## Binary Anchors

### Question serializer / state shape

`FUN_027e7410` decompiles with the following fields:

- `question`
- `name`
- `description`
- `fallbackOutcome`
- `namedOutcomes`
- `settledNamedOutcomes`

This confirms the question-level state shape is real in the binary, not just a
note artifact.

### Settlement/order guard path

`FUN_03eb9760` contains the live settled-order status strings:

- `outcomeSettledCanceled`
- `outcomeSettledRejected`

Those are confirmed to be in a large execution handler, not only in metadata
serde code.

### Outcome token validator

`FUN_031f1d70` validates raw outcome tokens and emits:

- `outcome token invalid decimals`
- `token is not outcome token`

Observed properties from decompile:

- it validates token id bounds
- it enforces outcome-token decimal policy
- it looks up the token through an outcome-specific index/map, not generic spot
  metadata alone
- on success it returns status `0x16f` and a structured record that includes a
  hardcoded `100000000` base constant alongside the validated token id and
  payload fields

Confirmed consequence:

- this is not just a string checker; it is the real raw outcome-token admission
  path that produces typed token metadata for later execution logic

Implication:
- there is a dedicated raw outcome-token layer beneath the ordinary spot-book
  layer

### Two-token outcome operation path

`FUN_03755b60` is a real execution path that calls `FUN_031f1d70` twice:

- once on `param_4`
- once on `param_4 + 0x20`

That means it is handling an operation over a pair of raw outcome tokens rather
than a single token or a pure question metadata object.

Observed properties from decompile:

- it first checks a separate outcome/question index before doing token
  validation
- it validates two raw outcome-token inputs through `FUN_031f1d70`
- it compares relationship fields extracted from both validated token records
- it returns a structured output record on success, or a structured error on
  mismatch
- it rejects one same-relation case before success with an explicit structured
  error at `DAT_00638e30` of length `0x1f`
- it rejects another capped-quantity case with a different structured error at
  `DAT_00638e22` of length `0x0e`
- it also enforces caps on two external quantity/config inputs:
  - `param_6[2] < 0x65`
  - `param_7[2] < 0x3e9`
  - `*(param_2 + 0x120) < 10000`

Most likely interpretation:

- this is one of the pairwise userOutcome paths (`MergeOutcome` or
  `NegateOutcome`)
- it is not a generic spot-book function

Current limitation:

- the exact user-facing variant name for `FUN_03755b60` is not yet proven

### Outcome execution anchor

`FUN_02114780` is confirmed to live in:

- `/home/ubuntu/hl/code_Mainnet/l1/src/exchange/impl_outcome.rs`

Observed consequence:

- the real outcome execution logic is localized in `impl_outcome.rs`
- the pairwise token helper is not an isolated library curiosity; it is wired
  into the exchange outcome handler

### Layer map inside `impl_outcome.rs`

The current binary evidence now supports a cleaner execution split:

- `FUN_03755b60`
  - raw two-token outcome primitive
  - validates both raw outcome tokens
  - enforces a token-pair relation
- `FUN_021113d0`
  - outcome-specific validator/updater above the raw primitive
  - checks per-asset state markers in exchange storage
  - derives a second asset relationship before user/account updates
- `FUN_02504870`
  - compact validator/normalizer over a derived asset pair plus amount
  - checks two mapped asset indices and parses a bounded amount
- `FUN_021179d0`
  - concrete accounting/update path above `FUN_02504870`
  - performs per-asset bounds checks and emits final records/events
- `FUN_0211cf80`
  - batch/finalizer over emitted records via a jump table
  - not a likely root-cause location for the core solvency bug

Implication:

- the likely question-level bug surface is above the raw pairwise primitive and
  below the final batch dispatcher
- the best current candidates are `FUN_021113d0` and `FUN_021179d0`, plus any
  `MergeQuestion`-specific branch in `FUN_02114780`

### Confirmed caller graph

The current xref graph is now concrete enough to narrow the execution stack:

- `FUN_02114780`
  - callers:
    - `02112fd2`
    - `021132d6`
- `FUN_021113d0`
  - callers:
    - `02110f3a` inside `FUN_021108c0`
    - `02115132` inside `FUN_02114780`
    - `02115325` inside `FUN_02114780`
    - `02117597` inside `FUN_021174b0`
- `FUN_021179d0`
  - callers:
    - `021151fa` inside `FUN_02114780`
    - `021153ed` inside `FUN_02114780`
    - `025b97ed` inside `FUN_025b0840`
- `FUN_02504870`
  - caller:
    - `02117a66` inside `FUN_021179d0`

Implication:

- `FUN_02114780` still looks like the main outcome entrypoint
- `FUN_021113d0` and `FUN_021179d0` are reused by adjacent callers, so the bug
  may live in a shared update layer rather than a single top-level action arm
- `FUN_02504870` looks narrow and downstream, which makes it more likely to be
  a validator/normalizer than the main source of a question-level solvency bug

Additional helper classification:

- `FUN_021108c0`
  - another stateful outcome helper
  - performs substantial exchange-state checks, calls `FUN_021113d0`, and also
    gates on success code `0x16f`
- `FUN_021174b0`
  - a smaller wrapper/helper that also feeds into `FUN_021113d0`
- `FUN_025b0840`
  - a much larger orchestrator that eventually calls `FUN_021174b0` and
    `FUN_021179d0`

Implication:

- the shared outcome-accounting path is reused outside the single
  `FUN_02114780` entrypoint
- if there is a collateral-conservation bug, it may be structural to the shared
  outcome-accounting helpers rather than unique to one top-level variant

### Confirmed branch structure inside `FUN_02114780`

Current disassembly confirms that `FUN_02114780` is not one flat path:

- one early branch enters through the computed call at `021147dc`
- later control flow splits into two sibling subpaths:
  - `02115132 -> FUN_021113d0`
  - `021151fa -> FUN_021179d0`
  - and separately:
  - `02115325 -> FUN_021113d0`
  - `021153ed -> FUN_021179d0`
- both branch families gate on success code `0x16f` before continuing

Implication:

- the main outcome handler appears to multiplex at least two distinct operation
  arms through the same validator/update helpers
- that makes a shared-accounting bug more plausible than a bug in one isolated
  parser or serializer branch

### Best current userOutcome mapping

The current binary evidence is now strong enough to narrow the operational
roles, but not yet strong enough to assign all four user-facing variant names
with certainty.

- `FUN_021108c0`
  - takes one external `ulong param_4`
  - performs substantial exchange-state checks
  - gates on success code `0x16f`
  - then calls `FUN_021113d0`
  - strongest current candidate for a single-outcome userOutcome arm
    (`SplitOutcome` or `MergeOutcome`)

- `FUN_02114780`
  - is still the strongest recovered top-level outcome entrypoint
  - is the only current top-level path with a direct computed call into
    `FUN_03755b60`
  - `FUN_03755b60` validates two raw outcome tokens and enforces a token-pair
    relation
  - both surviving sibling subpaths inside `FUN_02114780` then call:
    - `FUN_021113d0`
    - `FUN_021179d0`
    - `FUN_0211cf80`
  - strongest current candidate for the multi-token userOutcome arm
    (`NegateOutcome` or `MergeQuestion`)

- `FUN_021174b0`
  - takes two external asset ids from `param_4[0]` and `param_4[1]`
  - validates both ids
  - then directly calls `FUN_021113d0`
  - currently looks like a reusable two-asset wrapper below the top-level
    variant layer, not a confidently named top-level variant on its own

Current best mapping status:

- `SplitOutcome`
  - strongest current candidate: `FUN_021108c0`
- `MergeOutcome`
  - not yet isolated as a distinct top-level function
- `NegateOutcome`
  - strongest current candidate: the pairwise path through `FUN_02114780` and
    `FUN_03755b60`
- `MergeQuestion`
  - not yet isolated above `FUN_02114780`
  - if it is inside the current recovered stack, it is most likely one of the
    two sibling branch families already confirmed inside `FUN_02114780`

### Settlement-state gating evidence

Current direct evidence still points to order-level settlement gating, not a
recovered question-merge gate:

- `FUN_03eb9760` is still the confirmed home of:
  - `outcomeSettledCanceled`
  - `outcomeSettledRejected`
- `settledNamedOutcomes` still resolves only to the question serializer
  `FUN_027e7410`
- the currently recovered operational helpers:
  - `FUN_02114780`
  - `FUN_021108c0`
  - `FUN_021174b0`
  do not currently surface direct xrefs to settled-state strings in decompile

Implication:

- there is still no direct recovered evidence that `MergeQuestion` is gated by
  settled-question or settled-outcome state inside the current operational path
- that does **not** prove the gate is absent
- it does mean the gate has not yet been recovered from the current
  `impl_outcome.rs` execution stack

### Outcome action decode layer

Two more live binary functions are now confirmed:

- `FUN_03605b90`
- `FUN_036105c0`

Observed properties from decompile:

- both decode 0x40-byte action payloads
- both dispatch through jump tables:
  - `DAT_0062cbe0`
  - `DAT_0062cc90`
- both reference the same `fallbackOutcome` data string at `0x0062d21f`

Current interpretation:

- these are outcome-action decoders / enum dispatch layers
- they are not themselves the core solvency-critical merge logic
- they are useful because they show fallback-aware outcome actions are distinct
  at the decode layer before execution

Corrected interpretation:

- these decoder functions reference `fallbackOutcome` as a field name
- they should not be confused with the runtime policy guard for
  `Cannot trade fallback token`

## Raw Outcome Tokens vs Spot Books

The official asset-id docs say:

- outcome asset id = `100_000_000 + encoding`
- `encoding = 10 * outcome + side`
- example: `#10 = outcome 1, side 0`
- token name example: `+10`

That gives the question-1 mapping directly:

- outcome `10` -> `#100`, `#101`
- outcome `11` -> `#110`, `#111`
- outcome `12` -> `#120`, `#121`
- fallback outcome `13` -> `#130`, `#131`

Live testnet confirms those outcome-side assets exist through the ordinary info
API:

- `allMids` returns `#100`, `#101`, `#110`, `#111`, `#120`, `#121`, `#130`,
  `#131`
- `l2Book` for `#100` and `#101` returns a live complementary book:
  - `#100` best bid around `0.04`
  - `#101` best ask around `0.96`

Live mids for the four question-1 outcome pairs are:

- `#100 = 0.04`, `#101 = 0.96`
- `#110 = 0.8755`, `#111 = 0.1245`
- `#120 = 0.82625`, `#121 = 0.17375`
- `#130 = 0.5`, `#131 = 0.5`

Each pair sums to `1.0`, which is exactly what we would expect for a binary
YES/NO outcome pair.

Important correction:

- the `@100`, `@101`, ... entries from `spotMeta.universe` are a separate
  `@...` spot-book namespace
- the real outcome trading surface exposed by `allMids` / `l2Book` is the
  `#...` outcome-asset namespace from the official asset-id docs
- live `spotMeta` on testnet currently exposes `@...` synthetic spot books, but
  does not expose `#...` outcome-side assets or `+...` raw outcome tokens as
  ordinary spot metadata entries

So the earlier `@100/@101/...` interpretation was too coarse. The stronger
current model is:

- raw question/outcome state: `outcome`, `question`, `fallbackOutcome`,
  `namedOutcomes`, `settledNamedOutcomes`
- tradable outcome-side assets: `#encoding`
- separate ordinary spot-book namespace: `@index`

One economic signal now stands out:

- if `MergeQuestion` burns the first side across the four question-1 outcomes,
  then the basket `#100 + #110 + #120 + #130` is priced around `2.24175`
- the complementary basket `#101 + #111 + #121 + #131` is priced around
  `1.75825`

That is not proof of a bug by itself, but it shows the question-level merge
surface is economically nontrivial and worth treating as a real solvency
boundary.

## Fallback Guard

The testnet binary contains the exact string:

- `Cannot trade fallback token`

This confirms the fallback branch is treated specially somewhere in live logic.

Also confirmed:

- the literal `fallbackOutcome` field-name string at `0x0062d21f` currently
  xrefs only into:
  - `FUN_03605b90`
  - `FUN_036105c0`
- the `settledNamedOutcomes` string currently resolves only to the question
  serializer `FUN_027e7410`
- the `namedOutcomes` string currently resolves to:
  - `FUN_027e7410`
  - `FUN_0317b050`
  - `FUN_042cee40`
- those two functions are already classified as outcome-action decoders / enum
  dispatch layers, not the core economic path
- `FUN_0317b050` is still outcome-related, but current decompile evidence makes
  it look like schema/action registration plumbing rather than settlement or
  merge accounting:
  - `question`
  - `namedOutcome`
  - `quoteToken`
  - `tokens`
  - `registerOutcomeToken`
  - `registerNamedOutcome`
  - `changeOutcomeDescription`

What is not yet confirmed:

- which exact operation throws this guard
- whether it blocks ordinary trading, question-level merging, raw token
  conversion, or another outcome-specific path
- the string has still not been tied to a direct code xref, so it remains
  unresolved despite being present in the binary

## Strongest Current Bug Candidate

The best current candidate is a question-level reconciliation bug around
`MergeQuestion`, not a simple `SplitOutcome` / `MergeOutcome` bug.

Hypothesis:

- if settlement makes part of a question payable or retires named outcomes into
  `settledNamedOutcomes`
- and `MergeQuestion` still computes redeemability from stale active state
  without fully reconciling:
  - `fallbackOutcome`
  - `namedOutcomes`
  - `settledNamedOutcomes`
- then the same economic question state can release collateral twice

Updated narrowing:

- the current binary evidence pushes `fallbackOutcome` field-label handling into
  decoder/schema paths
- the current binary evidence also pushes `namedOutcomes` /
  `settledNamedOutcomes` field-label handling into serializer/schema paths
- that makes the most likely bug surface the stateful execution stack in
  `impl_outcome.rs`, especially:
  - `FUN_02114780`
  - `FUN_021113d0`
  - `FUN_021179d0`
- so the question is now less "where is the field named" and more "how is the
  fallback leg reconciled when question-level redemption runs"

The invariant that must hold:

- one complete question state must not return more than one collateral unit

Equivalently:

- settlement payout + merge payout must never exceed the collateral backing the
  same question state

## What Is Not Yet Proven

- the exact operational `MergeQuestion` handler
- the exact handler that throws `Cannot trade fallback token`
- the exact raw-token to spot-book mapping on testnet
- whether `Cannot trade fallback token` applies to user trading, user outcome
  actions, or only an internal path
- whether the candidate is exploitable before settlement, after settlement, or
  only on a fallback-resolution path

## Immediate Next Checks

1. Name the pairwise helper `FUN_03755b60` as `MergeOutcome` vs `NegateOutcome`.
2. Isolate the function that actually throws `Cannot trade fallback token`.
3. Recover the operational `MergeQuestion` execution path rather than only its
   enum/parser layers.
4. Determine whether `FUN_021113d0` or `FUN_021179d0` is specific to
   `MergeQuestion`, or whether both are reused by pairwise operations.
5. Confirm whether settlement retires mergeability or only orderability.
6. Recover the raw outcome token -> spot book mapping on testnet.
