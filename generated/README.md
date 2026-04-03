# Generated Docs and Artifacts

This directory holds generated outputs from reverse engineering, runtime extraction, and codebase analysis.

These files are useful, but they are not all equally curated. Think of this directory as the machine-fed artifact layer underneath the higher-level docs.

## Main Areas

- `re/` — binary profiling, coverage runs, auto-RE outputs, agent queues, coverage artifacts
- `runtime/` — runtime extraction outputs from validator logs and consensus logs
- `dot/` — generated graph sources
- dated markdown notes such as solvency reviews, runtime notes, or codebase maps

## Coverage State

Current saved RE coverage:

- `666 / 1470`
- `45.31%`

Primary references:

- [re/coverage/hl-node.latest.json](/Users/androolloyd/Development/hyperliquid-rust/docs/generated/re/coverage/hl-node.latest.json)
- [re/runs/2026-04-03T122729Z-agent-restart/summary.md](/Users/androolloyd/Development/hyperliquid-rust/docs/generated/re/runs/2026-04-03T122729Z-agent-restart/summary.md)

## How To Read This Directory

### If you want the latest binary coverage state

Start with:

- `re/coverage/hl-node.latest.json`
- the newest run under `re/runs/`

### If you want current runtime truth

Start with:

- `runtime/`
- `hl-re runtime-report` outputs referenced in the run summaries

### If you want subsystem call graphs or structure maps

Start with:

- `dot/`
- generated markdown notes linked from the docs hub and paper pages

## Relationship To The Rest Of The Repo

- `knowledge/` is the structured claims/source layer
- `docs/generated/` is the artifact layer
- `docs/` is the curated presentation layer

The intended flow is:

`artifact -> claim -> compiled doc -> implementation/test task`
