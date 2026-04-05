# Open Claims

Generated from `knowledge/claims` by `tools/compile_knowledge_docs.py`.

- `claim.bridge_finalization_and_validator_set_flow` - Bridge2 stages withdrawals and validator-set updates through signatures plus finalized-vote state (`bridge`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.hyperbft_topology_and_rotation` - HyperBFT uses broadcaster ingress, validator/sentry admission checks, and RoundRobinTtl proposer rotation (`consensus`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.validator_lifecycle_and_jail_flow` - Validator lifecycle is time-epoch gated, jail-aware, and signer-checked against current epoch state (`consensus`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.aligned_quote_token_sampling` - Aligned quote token state samples validator risk-free-rate votes in begin-block effect 9 (`execution`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.corewriter_delayed_action_surface` - CoreWriter re-enters L1 through a delayed action lane with 15 known action types (`execution`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.action_surface_testnet` - Current widened testnet action surface is 97 variants with 126 sub-types (`execution`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`execution`, `confirmed`, `confirmed`, `local_impl`)
- `claim.fill_hash_struct_surface` - Fill hashes use a separate 18-field payload and API wrappers add coin and feeToken (`hashing`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.resphash_backend_split` - RespHash backends are chain-scoped, but mainnet serialization is not fully wired (`hashing`, `confirmed`, `confirmed`, `local_impl`)
- `claim.liquidation_scope_boundaries` - Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes (`liquidation`, `confirmed`, `confirmed`, `local_impl`)
- `claim.portfolio_margin_liquidation_mode` - Portfolio margin liquidates as a generalized cross-margin account (`liquidation`, `confirmed`, `confirmed`, `protocol`)
- `claim.outcome_settlement_and_question_surface` - Outcomes are 1x isolated settlement markets with explicit question and fallback metadata (`outcomes`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`state`, `confirmed`, `confirmed`, `testnet_impl`)
- `claim.action_delayer_maturity_surface` - ActionDelayer drain is a named begin-block lane, while mode semantics remain partly open (`execution`, `implemented`, `confirmed`, `local_impl`)
- `claim.outcome_merge_question_fallback_risk` - Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk (`outcomes`, `observed`, `hypothesis`, `testnet_impl`)
