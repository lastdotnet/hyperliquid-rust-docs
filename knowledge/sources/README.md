# Sources

Source records normalize where claims came from.

Typical source kinds:

- `official_doc`
- `local_doc`
- `generated_note`
- `binary_re`
- `runtime_log`
- `api_capture`
- `code`

Each source should have:

- a stable `id`
- a `kind`
- a `scope`
- either a local `path` or remote `url`
- a short summary of why it matters
