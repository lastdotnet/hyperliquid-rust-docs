# Truth Register

Generated from `knowledge/claims` by `tools/compile_knowledge_docs.py`.

This is the compact fact ledger for the repo. Treat it as the fastest way to answer:

- what is currently confirmed
- what is still active or unresolved
- which research lanes are still only inferred or hypothesis-grade

## Confirmed Facts

- `claim.bridge_finalization_and_validator_set_flow` - Bridge2 stages withdrawals and validator-set updates through signatures plus finalized-vote state (`bridge`, `testnet_impl`, `active`)
- `claim.hyperbft_topology_and_rotation` - HyperBFT uses broadcaster ingress, validator/sentry admission checks, and RoundRobinTtl proposer rotation (`consensus`, `testnet_impl`, `active`)
- `claim.validator_lifecycle_and_jail_flow` - Validator lifecycle is time-epoch gated, jail-aware, and signer-checked against current epoch state (`consensus`, `testnet_impl`, `active`)
- `claim.action_delayer_maturity_surface` - ActionDelayer drain is a named begin-block lane, while mode semantics remain partly open (`execution`, `local_impl`, `active`)
- `claim.aligned_quote_token_sampling` - Aligned quote token state samples validator risk-free-rate votes in begin-block effect 9 (`execution`, `testnet_impl`, `active`)
- `claim.begin_block_ordering_surface` - Begin-block ordering is 9 named effects with BOLE at #3 and ActionDelayer at #8 (`execution`, `testnet_impl`, `closed`)
- `claim.corewriter_delayed_action_surface` - CoreWriter re-enters L1 through a delayed action lane with 15 known action types (`execution`, `testnet_impl`, `active`)
- `claim.action_surface_testnet` - Current widened testnet action surface is 97 variants with 126 sub-types (`execution`, `testnet_impl`, `active`)
- `claim.begin_block_bole_ordering_conflict` - Stale BOLE begin-block ordering conflict is resolved (`execution`, `testnet_impl`, `closed`)
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`execution`, `local_impl`, `open`)
- `claim.fill_hash_struct_surface` - Fill hashes use a separate 18-field payload and API wrappers add coin and feeToken (`hashing`, `testnet_impl`, `active`)
- `claim.serializer_g_success_error_split` - G-family action hashes use H-shape success and G-shape error payloads (`hashing`, `testnet_impl`, `closed`)
- `claim.resphash_backend_split` - RespHash backends are chain-scoped, but mainnet serialization is not fully wired (`hashing`, `local_impl`, `open`)
- `claim.liquidation_scope_boundaries` - Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes (`liquidation`, `local_impl`, `open`)
- `claim.portfolio_margin_liquidation_mode` - Portfolio margin liquidates as a generalized cross-margin account (`liquidation`, `protocol`, `active`)
- `claim.outcome_settlement_and_question_surface` - Outcomes are 1x isolated settlement markets with explicit question and fallback metadata (`outcomes`, `testnet_impl`, `active`)
- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`state`, `testnet_impl`, `open`)

## Active Facts

- `claim.bridge_finalization_and_validator_set_flow` - Bridge2 stages withdrawals and validator-set updates through signatures plus finalized-vote state (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.hyperbft_topology_and_rotation` - HyperBFT uses broadcaster ingress, validator/sentry admission checks, and RoundRobinTtl proposer rotation (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.validator_lifecycle_and_jail_flow` - Validator lifecycle is time-epoch gated, jail-aware, and signer-checked against current epoch state (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.action_delayer_maturity_surface` - ActionDelayer drain is a named begin-block lane, while mode semantics remain partly open (`implemented`, `confirmed`, `local_impl`)
- `claim.aligned_quote_token_sampling` - Aligned quote token state samples validator risk-free-rate votes in begin-block effect 9 (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.corewriter_delayed_action_surface` - CoreWriter re-enters L1 through a delayed action lane with 15 known action types (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.action_surface_testnet` - Current widened testnet action surface is 97 variants with 126 sub-types (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`confirmed`, `confirmed`, `local_impl`)
- `claim.fill_hash_struct_surface` - Fill hashes use a separate 18-field payload and API wrappers add coin and feeToken (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.resphash_backend_split` - RespHash backends are chain-scoped, but mainnet serialization is not fully wired (`confirmed`, `confirmed`, `local_impl`)
- `claim.liquidation_scope_boundaries` - Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes (`confirmed`, `confirmed`, `local_impl`)
- `claim.portfolio_margin_liquidation_mode` - Portfolio margin liquidates as a generalized cross-margin account (`confirmed`, `confirmed`, `protocol`)
- `claim.outcome_merge_question_fallback_risk` - Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk (`observed`, `hypothesis`, `testnet_impl`)
- `claim.outcome_settlement_and_question_surface` - Outcomes are 1x isolated settlement markets with explicit question and fallback metadata (`confirmed`, `confirmed`, `testnet_impl`)
- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`confirmed`, `confirmed`, `testnet_impl`)

## Inferred Or Hypothesis Lanes

- `claim.outcome_merge_question_fallback_risk` - Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk (`observed`, `hypothesis`, `testnet_impl`, `active`)
