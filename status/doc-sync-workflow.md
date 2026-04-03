# Doc Sync Workflow

This repo needs a repeatable way to keep docs, claims, and code comments aligned as reverse-engineering and implementation move.

The intended loop is:

1. discover a new protocol or implementation truth
2. promote it into `knowledge/claims`
3. attach or update `knowledge/sources`
4. rerun `python3 tools/compile_knowledge_docs.py`
5. inspect [protocol-sync-report.md](/Users/androolloyd/Development/hyperliquid-rust/docs/status/protocol-sync-report.md)
6. fix drift in docs, task notes, comments, or code

## What Belongs In Claims

Put a claim in `knowledge/claims/` when the finding changes something structural, for example:

- state cardinality
- execution ordering
- field names or field order
- action schema or signing domain
- chain-specific implementation differences

## What Belongs In Sources

Every claim should point at the strongest nearby evidence:

- official doc
- local note
- generated RE note
- runtime artifact
- code path

## Current Example

The active seeded example is:

- [claim.exchange_struct_cardinality](/Users/androolloyd/Development/hyperliquid-rust/knowledge/claims/exchange-struct-cardinality.md)

That claim currently asserts:

- `Exchange` cardinality is `57`

The sync report checks selected repo surfaces for field-count drift against that governing claim.

## Rule Of Thumb

- update the claim first
- regenerate the report
- then repair the drift

That keeps the repo from accumulating stale prose like `56 fields` in one place and `57 fields` in another.
