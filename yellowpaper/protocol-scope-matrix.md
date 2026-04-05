# Protocol Scope Matrix

Generated from `knowledge/claims` by `tools/compile_knowledge_docs.py`.

## local_impl

- `claim.action_delayer_maturity_surface` - ActionDelayer drain is a named begin-block lane, while mode semantics remain partly open (`execution`, `implemented`, `active`)
- `claim.liquidation_scope_boundaries` - Liquidation family boundaries are now explicit across perps, PM, BOLE, spot, and outcomes (`liquidation`, `confirmed`, `open`)
- `claim.resphash_backend_split` - RespHash backends are chain-scoped, but mainnet serialization is not fully wired (`hashing`, `confirmed`, `open`)
- `claim.block_lifecycle_phase_map` - The repo now uses one explicit block lifecycle phase map and hook surface (`execution`, `confirmed`, `open`)

## protocol

- `claim.portfolio_margin_liquidation_mode` - Portfolio margin liquidates as a generalized cross-margin account (`liquidation`, `confirmed`, `active`)

## testnet_impl

- `claim.aligned_quote_token_sampling` - Aligned quote token state samples validator risk-free-rate votes in begin-block effect 9 (`execution`, `confirmed`, `active`)
- `claim.begin_block_ordering_surface` - Begin-block ordering is 9 named effects with BOLE at #3 and ActionDelayer at #8 (`execution`, `confirmed`, `closed`)
- `claim.bridge_finalization_and_validator_set_flow` - Bridge2 stages withdrawals and validator-set updates through signatures plus finalized-vote state (`bridge`, `confirmed`, `active`)
- `claim.corewriter_delayed_action_surface` - CoreWriter re-enters L1 through a delayed action lane with 15 known action types (`execution`, `confirmed`, `active`)
- `claim.action_surface_testnet` - Current widened testnet action surface is 97 variants with 126 sub-types (`execution`, `confirmed`, `active`)
- `claim.exchange_struct_cardinality` - Exchange struct cardinality is currently 57 fields (`state`, `confirmed`, `open`)
- `claim.fill_hash_struct_surface` - Fill hashes use a separate 18-field payload and API wrappers add coin and feeToken (`hashing`, `confirmed`, `active`)
- `claim.serializer_g_success_error_split` - G-family action hashes use H-shape success and G-shape error payloads (`hashing`, `confirmed`, `closed`)
- `claim.hyperbft_topology_and_rotation` - HyperBFT uses broadcaster ingress, validator/sentry admission checks, and RoundRobinTtl proposer rotation (`consensus`, `confirmed`, `active`)
- `claim.outcome_merge_question_fallback_risk` - Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk (`outcomes`, `observed`, `active`)
- `claim.outcome_settlement_and_question_surface` - Outcomes are 1x isolated settlement markets with explicit question and fallback metadata (`outcomes`, `confirmed`, `active`)
- `claim.begin_block_bole_ordering_conflict` - Stale BOLE begin-block ordering conflict is resolved (`execution`, `confirmed`, `closed`)
- `claim.validator_lifecycle_and_jail_flow` - Validator lifecycle is time-epoch gated, jail-aware, and signer-checked against current epoch state (`consensus`, `confirmed`, `active`)
