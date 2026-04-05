# Claims Index

Generated from `knowledge/claims` by `tools/compile_knowledge_docs.py`.

## bridge

- `claim.bridge_finalization_and_validator_set_flow` - Bridge2 stages withdrawals and validator-set updates through signatures plus finalized-vote state (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/bridge-finalization-and-validator-set-flow.md`

## consensus

- `claim.hyperbft_topology_and_rotation` - HyperBFT uses broadcaster ingress, validator/sentry admission checks, and RoundRobinTtl proposer rotation (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/hyperbft-topology-and-rotation.md`
- `claim.validator_lifecycle_and_jail_flow` - Validator lifecycle is time-epoch gated, jail-aware, and signer-checked against current epoch state (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/validator-lifecycle-and-jail-flow.md`

## execution

- `claim.action_delayer_maturity_surface` - ActionDelayer drain is a named begin-block lane, while mode semantics remain partly open (`implemented`, `confirmed`, `local_impl`, `active`)
  - Source file: `knowledge/claims/action-delayer-maturity-surface.md`
- `claim.aligned_quote_token_sampling` - Aligned quote token state samples validator risk-free-rate votes in begin-block effect 9 (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/aligned-quote-token-sampling.md`
- `claim.begin_block_ordering_surface` - Begin-block ordering is 9 named effects with BOLE at #3 and ActionDelayer at #8 (`confirmed`, `confirmed`, `testnet_impl`, `closed`)
  - Source file: `knowledge/claims/begin-block-ordering-surface.md`
- `claim.corewriter_delayed_action_surface` - CoreWriter re-enters L1 through a delayed action lane with 15 known action types (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/corewriter-delayed-action-surface.md`
- `claim.action_surface_testnet` - Current widened testnet action surface is 97 variants with 126 sub-types (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/action-surface-testnet.md`
- `claim.begin_block_bole_ordering_conflict` - Stale BOLE begin-block ordering conflict is resolved (`confirmed`, `confirmed`, `testnet_impl`, `closed`)
  - Source file: `knowledge/claims/begin-block-bole-ordering-conflict.md`
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`confirmed`, `confirmed`, `local_impl`, `open`)
  - Source file: `knowledge/claims/block-lifecycle-phase-map.md`

## hashing

- `claim.fill_hash_struct_surface` - Fill hashes use a separate 18-field payload and API wrappers add coin and feeToken (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/fill-hash-struct-surface.md`
- `claim.serializer_g_success_error_split` - G-family action hashes use H-shape success and G-shape error payloads (`confirmed`, `confirmed`, `testnet_impl`, `closed`)
  - Source file: `knowledge/claims/serializer-g-success-error-split.md`
- `claim.resphash_backend_split` - RespHash backends are chain-scoped, but mainnet serialization is not fully wired (`confirmed`, `confirmed`, `local_impl`, `open`)
  - Source file: `knowledge/claims/resphash-backend-split.md`

## liquidation

- `claim.liquidation_scope_boundaries` - Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes (`confirmed`, `confirmed`, `local_impl`, `open`)
  - Source file: `knowledge/claims/liquidation-scope-boundaries.md`
- `claim.portfolio_margin_liquidation_mode` - Portfolio margin liquidates as a generalized cross-margin account (`confirmed`, `confirmed`, `protocol`, `active`)
  - Source file: `knowledge/claims/portfolio-margin-liquidation-mode.md`

## outcomes

- `claim.outcome_merge_question_fallback_risk` - Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk (`observed`, `hypothesis`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/outcome-merge-question-fallback-risk.md`
- `claim.outcome_settlement_and_question_surface` - Outcomes are 1x isolated settlement markets with explicit question and fallback metadata (`confirmed`, `confirmed`, `testnet_impl`, `active`)
  - Source file: `knowledge/claims/outcome-settlement-and-question-surface.md`

## state

- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`confirmed`, `confirmed`, `testnet_impl`, `open`)
  - Source file: `knowledge/claims/exchange-struct-cardinality.md`
