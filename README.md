# Local LLM + Claude Code Guide

### 📖 Read it here → **[jaloliddin-rus.github.io/local-llm-claude-code-guide](https://jaloliddin-rus.github.io/local-llm-claude-code-guide/)**

A step-by-step guide to running **Qwen 3.6** and **Gemma 4** locally on a 24 GB GPU, with **Claude Code** talking to them over SSH. Every number and config in this guide was measured on real hardware (RTX A5000 24 GB, Linux Mint 22.3, ECC disabled), not estimated.

---

## What you'll end up with

Three local models, each tuned for its strength:

| Model | Generation speed | Use for |
|---|---|---|
| **Qwen 3.6 27B** | ~25 tok/s | Claude Code agentic coding |
| **Gemma 4 26B-A4B (MoE)** | **~102 tok/s** | Fast daily writing, brainstorming |
| **Gemma 4 31B (dense)** | ~29 tok/s | Highest-quality writing, analysis |

Six shell functions on the Linux box to start/switch models. Two shell functions on the Mac to toggle between local and cloud Claude. All on one port. Swap models in ~10 seconds.

---

## Why this exists

Most "run an LLM locally" guides stop at `llama.cpp --help`. This one goes the whole way: Unsloth Dynamic 2.0 quants, exact KV-cache settings per model, ECC tricks, SSH tunnel, Claude Code env vars, and the privacy flags that actually matter. It's what a full setup looks like when you care about both performance and privacy.

---

## Repo layout

This is a [Jekyll](https://jekyllrb.com/) site deployed to GitHub Pages using the [just-the-docs](https://just-the-docs.com/) theme.

- `index.md` — landing page
- `01-hardware.md` … `14-appendix.md` — one page per guide section
- `_config.yml` — Jekyll config (theme, search, callouts)
- `_sass/` — custom dark/light color schemes, typography, UI polish
- `_includes/head_custom.html` — Google Fonts (Inter + JetBrains Mono)
- `_includes/header_custom.html` — theme toggle button + localStorage JS
- `assets/css/` — entry-point stylesheets for each color scheme

---

## Read the Markdown directly

If you just want the raw source without the site styling, every section of the guide is a plain `.md` file at the root of this repo — start with `index.md` and follow the numbered files in order.

---

## Contributing

Found a bug, have better numbers on a different card, or know a trick that should be in here? Open an issue or PR.
