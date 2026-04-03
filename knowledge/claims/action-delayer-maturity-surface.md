---
id: claim.action_delayer_maturity_surface
title: ActionDelayer drain is a named begin-block lane, while mode semantics remain partly open
subsystem: execution
promotion: implemented
status: active
confidence: confirmed
scope: local_impl
sources: ["source.action_delayer_note", "source.l1_evm_bridge_flow_note", "source.exchange-state-note", "source.block_lifecycle_note"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The currently confirmed ActionDelayer surface is narrower than the full control
plane, but broader than just queue timestamps. Queued delayed actions carry
explicit `matured_at` timestamps, drain-time execution is keyed off those stored
timestamps, and the matured drain is a named begin-block lane. The exact
runtime semantics of `enabled`, `delayer_mode`, `status_guard`, and
`vty_trackers` remain partly open.

## Evidence

- [`docs/obsidian/Action Delayer.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Action%20Delayer.md) now records the confirmed queue shape and the current maturity-flow boundary.
- [`docs/obsidian/L1 EVM Bridge Flow.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/L1%20EVM%20Bridge%20Flow.md) confirms CoreWriter actions enqueue and later execute from the delayed queue during `begin_block`.
- [`docs/obsidian/Exchange State.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Exchange%20State.md) pins `action_delayer` as a stable Exchange field with the 9 confirmed sub-fields.
- [`docs/obsidian/Block Lifecycle.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/obsidian/Block%20Lifecycle.md) places delayed CoreWriter execution in begin-block effect 8.

## Open Questions

- Whether `enabled == false` means bypass delay, reject new delayed actions, or only disable status-mode updates.
- Whether `vty_trackers` / `status_guard` / `delayer_mode` affect enqueue-time delay selection, status transitions, or both.

## Implementation Impact

- Honor stored delayed-action maturity at drain time even when the delayer is disabled, so queued entries do not black-hole.
- Keep `enabled` / mode semantics explicit and unguessed until stronger RE closes them.
