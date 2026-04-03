# Claims Index

Generated from `knowledge/claims` by `tools/compile_knowledge_docs.py`.

## execution

- `claim.action_delayer_maturity_surface` - ActionDelayer maturity is queue-entry driven, while mode semantics remain partly open (`implemented`, `confirmed`, `local_impl`, `active`)
  - Source file: `knowledge/claims/action-delayer-maturity-surface.md`
- `claim.begin_block_ordering_surface` - Begin-block ordering is partly confirmed, but BOLE and delayed-action placement still need closure (`confirmed`, `inferred`, `local_impl`, `open`)
  - Source file: `knowledge/claims/begin-block-ordering-surface.md`
- `claim.begin_block_bole_ordering_conflict` - Local begin-block notes disagree on where BOLE liquidations run (`observed`, `inferred`, `local_impl`, `open`)
  - Source file: `knowledge/claims/begin-block-bole-ordering-conflict.md`
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`confirmed`, `confirmed`, `local_impl`, `open`)
  - Source file: `knowledge/claims/block-lifecycle-phase-map.md`

## hashing

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

## state

- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`confirmed`, `confirmed`, `testnet_impl`, `open`)
  - Source file: `knowledge/claims/exchange-struct-cardinality.md`
