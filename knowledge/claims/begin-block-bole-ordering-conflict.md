---
id: claim.begin_block_bole_ordering_conflict
title: Local begin-block notes disagree on where BOLE liquidations run
subsystem: execution
promotion: observed
status: open
confidence: inferred
scope: local_impl
sources: ["source.abci_state_machine_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The repo currently contains conflicting local claims about BOLE liquidation
position inside `begin_block`: one path says effect 3, another says effect 9.

## Evidence

- [`docs/obsidian/ABCI State Machine.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/ABCI%20State%20Machine.md) describes BOLE as effect 3.
- Local engine comments and enum ordering in the ADL worktree currently place it
  as effect 9.

## Open Questions

- Which ordering is directly confirmed by the latest binary trace.
- Whether BOLE should be fully executed at top-of-block or split into a top-of-
  block evaluation plus later execution.

## Implementation Impact

- Do not spread BOLE liquidation hooks across multiple block phases.
- Route it through one explicit begin-block effect once the ordering is settled.
