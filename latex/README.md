# LaTeX Papers

This directory holds typeset paper sources for the public docs set.

Current documents:

- `whitepaper.tex` — narrative architecture white paper
- `yellowpaper.tex` — generated protocol-reference yellow paper from `docs/yellowpaper/index.md`

Build with one of:

```bash
tectonic whitepaper.tex
tectonic yellowpaper.tex
```

or:

```bash
pdflatex whitepaper.tex
pdflatex whitepaper.tex
pdflatex yellowpaper.tex
pdflatex yellowpaper.tex
```

From the repo root, use:

```bash
tools/build_latex_docs.sh
```

That helper will use `tectonic`, `pdflatex`, or `lualatex` when available and
print a clear error if no TeX toolchain is installed locally. The helper also
regenerates `yellowpaper.tex` from the markdown source before building.
