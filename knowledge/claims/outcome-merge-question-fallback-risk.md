---
id: claim.outcome_merge_question_fallback_risk
title: Multi-outcome MergeQuestion fallback reconciliation is the leading outcome solvency risk
subsystem: outcomes
promotion: observed
status: active
confidence: hypothesis
scope: testnet_impl
sources: ["source.outcome_solvency_review_2026_04_03"]
docs_targets: ["yellowpaper", "findings", "status"]
updated: 2026-04-03
---

## Summary

The strongest current solvency hypothesis is that multi-outcome question-level
merge and settlement reconciliation can mishandle fallback legs and settled
named outcomes.

## Evidence

- Raw pairwise outcome token helpers look more constrained than the shared
  question-level execution path.
- Field-label xrefs for `fallbackOutcome` and `settledNamedOutcomes` mostly land
  in serializer or decoder paths, not the core economic path.
- Testnet outcome metadata exposes a real fallback outcome alongside named
  outcomes, making the reconciliation boundary economically meaningful.

## Open Questions

- Exact `MergeQuestion` branch identity in the binary.
- Exact operational guard for `Cannot trade fallback token`.
- Whether settlement retires mergeability or only orderability.

## Implementation Impact

- Keep outcome solvency work separated from ordinary perp or spot liquidation.
- Treat question-level accounting as the highest-risk implementation area.
