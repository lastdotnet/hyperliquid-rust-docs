# Claims

Each claim is a small markdown record with simple frontmatter.

Required fields:

- `id`
- `title`
- `subsystem`
- `promotion`
- `status`
- `confidence`
- `scope`
- `sources`
- `docs_targets`
- `updated`

Recommended body sections:

- `Summary`
- `Evidence`
- `Open Questions`
- `Implementation Impact`

The compiler reads these files and generates claim indexes for:

- [`docs/findings/claims-index.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/findings/claims-index.md)
- [`docs/status/open-claims.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/status/open-claims.md)
- [`docs/yellowpaper/protocol-scope-matrix.md`](/Users/androolloyd/Development/hyperliquid-rust/docs/yellowpaper/protocol-scope-matrix.md)
