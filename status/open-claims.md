# Open Claims

Generated from `knowledge/claims` by `tools/compile_knowledge_docs.py`.

- `claim.begin_block_ordering_surface` - Begin-block ordering is partly confirmed, but BOLE and delayed-action placement still need closure (`execution`, `confirmed`, `inferred`, `local_impl`)
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`execution`, `confirmed`, `confirmed`, `local_impl`)
- `claim.resphash_backend_split` - RespHash backends are chain-scoped, but mainnet serialization is not fully wired (`hashing`, `confirmed`, `confirmed`, `local_impl`)
- `claim.liquidation_scope_boundaries` - Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes (`liquidation`, `confirmed`, `confirmed`, `local_impl`)
- `claim.portfolio_margin_liquidation_mode` - Portfolio margin liquidates as a generalized cross-margin account (`liquidation`, `confirmed`, `confirmed`, `protocol`)
- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`state`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.action_delayer_maturity_surface` - ActionDelayer maturity is queue-entry driven, while mode semantics remain partly open (`execution`, `implemented`, `confirmed`, `local_impl`)
- `claim.begin_block_bole_ordering_conflict` - Local begin-block notes disagree on where BOLE liquidations run (`execution`, `observed`, `inferred`, `local_impl`)
- `claim.outcome_merge_question_fallback_risk` - Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk (`outcomes`, `observed`, `hypothesis`, `testnet_impl`)
