# Knowledge Layer

This directory is the structured source-of-truth layer for the docs system.

The intended flow is:

1. ingest raw evidence
2. promote evidence into claims with provenance
3. compile claims into docs, findings, and status pages
4. implement or verify against those claims in code

Run `python3 tools/compile_knowledge_docs.py` after changing claims or sources.
That compiler now also emits a repo-facing sync check at
[`docs/status/protocol-sync-report.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/status/protocol-sync-report.md).

Directory layout:

- `claims/` - structured protocol and implementation claims
- `sources/` - normalized records for docs, binaries, logs, APIs, and notes

Promotion states:

- `observed` - seen in raw evidence
- `confirmed` - corroborated by multiple sources or strong primary evidence
- `implemented` - wired into the local codebase
- `verified` - tested or replay-confirmed

Scope labels:

- `protocol`
- `mainnet_impl`
- `testnet_impl`
- `local_impl`

Confidence labels:

- `confirmed`
- `inferred`
- `hypothesis`
