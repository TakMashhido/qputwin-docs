---
title: Diagrams and figures
description: QPUTwin's architecture diagrams, rebuilding the publication-ready benchmark figures and this MkDocs documentation site.
---

# Diagrams & figures

## Architecture diagrams

The sixteen architecture, process and data diagrams used by this site and the README are pre-rendered SVGs in `assets/diagrams/` of this site. Each embeds the subset of fonts it uses, so it renders identically as an `<img>` on this site and on GitHub.

## Benchmark figures

```bash
uv run --group bench python benchmarks/run_benchmarks.py all   # data → benchmarks/results/*.json
uv run --group bench python benchmarks/plot_figures.py         # figures → benchmarks/figures/
```

Copy the regenerated SVG and PNG files into `assets/figures/` of the [qputwin-docs](https://github.com/TakMashhido/qputwin-docs) repository to update this site.

Figure conventions:

* **Formats:** vector PDF (TrueType-embedded) and SVG, plus a 300-dpi PNG; a CSV of exactly the plotted values in `benchmarks/figures/data/`.
* **Sizes:** 3.5 in (single column) or 7.2 in (double column); DejaVu Sans at 7–8.5 pt.
* **Colour:** the seven devices take the first seven slots of a categorical palette validated for colour-vision deficiency, in fixed order (adjacent-pair CVD ΔE ≥ 9.1, normal-vision ΔE ≥ 19.6 on white). Every series also has its own marker and line style, so the figures survive grayscale printing.
* **Layout:** a single y-axis per panel, thin solid gridlines, sparse direct labels.

## This website

The source of this website is the [qputwin-docs](https://github.com/TakMashhido/qputwin-docs) repository. Every push to its `main` branch rebuilds and publishes it at <https://koustav.isroot.in/qputwin-docs/>. To preview it locally:

```bash
pip install -r requirements.txt
mkdocs serve            # http://127.0.0.1:8088 with live reload
mkdocs build --strict   # static site in ./site, failing on broken links
```

The site uses MkDocs Material with MathJax for equations. It is multipage, searchable, and supports light and dark themes.
