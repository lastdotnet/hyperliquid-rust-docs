# Claim Schema

Frontmatter uses simple `key: value` lines. Lists should use JSON array syntax.

Example:

```text
---
id: claim.example
title: Example claim title
subsystem: execution
promotion: observed
status: open
confidence: inferred
scope: protocol
sources: ["source.example_a", "source.example_b"]
docs_targets: ["whitepaper", "yellowpaper", "status"]
updated: 2026-04-03
---
```

Field meanings:

- `id`
  - stable claim identifier
- `title`
  - short human-readable statement
- `subsystem`
  - example: `liquidation`, `consensus`, `hashing`, `outcomes`
- `promotion`
  - `observed`, `confirmed`, `implemented`, `verified`
- `status`
  - `open`, `active`, `closed`
- `confidence`
  - `confirmed`, `inferred`, `hypothesis`
- `scope`
  - `protocol`, `mainnet_impl`, `testnet_impl`, `local_impl`
- `sources`
  - source record IDs
- `docs_targets`
  - where the claim should appear
- `updated`
  - ISO date
