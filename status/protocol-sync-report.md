# Protocol Sync Report

Generated from `knowledge/claims` and selected repo surfaces by `tools/compile_knowledge_docs.py`.

- expected Exchange field count: `57`
- aligned references: `4`
- drift references: `0`

## Workflow

1. update the governing claim when new RE changes protocol or implementation truth
2. rerun `python3 tools/compile_knowledge_docs.py`
3. inspect this report
4. fix drift in docs, comments, task notes, or code

## Drift

- none

## Aligned References

- `docs/obsidian/Exchange State.md:3` — The central state object of [[Hyperliquid]] — **57 fields** (binary says "struct Exchange with 57 elements") that contain ALL L1 state. Every block's `begin_block -> process actions -> end_block` mutates this struct.
- `docs/obsidian/Exchange State.md:11` — Verified from binary rodata at 0x386138 (and 7 duplicate copies). Field names are concatenated in exact serialization order. Binary descriptor: `struct Exchange with 57 elements`.
- `docs/obsidian/ABCI State Machine.md:66` — The `Exchange` struct is the god object ("struct Exchange with 57 elements" in binary). Uses **full field names** in serde (NOT 2-char codes). See [[Exchange State]] for complete 57-field map. Key field categories:
- `crates/hl-engine/src/bootstrap_fast.rs:219` — // Exchange has 57 fields (binary: "struct Exchange with 57 elements").
