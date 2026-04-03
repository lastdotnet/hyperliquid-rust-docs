# Paper Build Workflow

This repo now has a paper set, not just a loose pile of markdown files.

The intended source-of-truth order is:

`sources -> claims -> compiled indexes -> paper markdown -> rendered HTML -> public export`

## Surfaces

- white paper narrative:
  - [docs/whitepaper/index.md](/Users/androolloyd/Development/hyperliquid-rust/docs/whitepaper/index.md)
  - [docs/whitepaper/index.html](/Users/androolloyd/Development/hyperliquid-rust/docs/whitepaper/index.html)
  - [docs/latex/whitepaper.tex](/Users/androolloyd/Development/hyperliquid-rust/docs/latex/whitepaper.tex)
- yellow paper:
  - [docs/yellowpaper/index.md](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/index.md)
  - [docs/yellowpaper/index.html](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/index.html)
- action inventory:
  - [docs/yellowpaper/action-inventory.md](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/action-inventory.md)
  - [docs/yellowpaper/action-inventory.html](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/action-inventory.html)
- fact dashboards:
  - [docs/findings/truth-register.md](/Users/androolloyd/Development/hyperliquid-rust/docs/findings/truth-register.md)
  - [docs/findings/truth-register.html](/Users/androolloyd/Development/hyperliquid-rust/docs/findings/truth-register.html)
  - [docs/status/open-claims.md](/Users/androolloyd/Development/hyperliquid-rust/docs/status/open-claims.md)
  - [docs/status/open-claims.html](/Users/androolloyd/Development/hyperliquid-rust/docs/status/open-claims.html)

## Rules

1. New protocol truth starts in:
   - [knowledge/sources/](/Users/androolloyd/Development/hyperliquid-rust/knowledge/sources)
   - [knowledge/claims/](/Users/androolloyd/Development/hyperliquid-rust/knowledge/claims)
2. Compiled fact surfaces come from:
   - [tools/compile_knowledge_docs.py](/Users/androolloyd/Development/hyperliquid-rust/tools/compile_knowledge_docs.py)
3. Narrative paper text should summarize claims, not silently compete with them.
4. When a paper sentence changes repo truth, update the claim first or in the same pass.

## Commands

Local paper build:

```bash
tools/build_papers.sh
```

This does:

1. compile the knowledge indexes
2. validate the rendered HTML paper shells
3. attempt a local LaTeX build

Public publish:

```bash
tools/publish_public_docs.sh
```

## Fact-First Maintenance

Before editing a paper chapter, check:

- [docs/findings/truth-register.html](/Users/androolloyd/Development/hyperliquid-rust/docs/findings/truth-register.html)
- [docs/status/open-claims.html](/Users/androolloyd/Development/hyperliquid-rust/docs/status/open-claims.html)
- [docs/yellowpaper/protocol-scope-matrix.md](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/protocol-scope-matrix.md)

That keeps the paper set aligned with the current research instead of drifting
into repeated or conflicting statements.
