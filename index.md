---
title: Home
layout: default
nav_order: 1
---

<section class="hero">
  <h1>Local LLM + Claude Code</h1>
  <p class="hero-subtitle">Run Qwen 3.6 and Gemma 4 locally on a 24&nbsp;GB GPU. Talk to them from Claude Code over SSH. Full privacy, zero marginal cost.</p>
  <div class="hero-buttons">
    <a href="./01-hardware.html" class="btn btn-purple fs-5">Get started →</a>
    <a href="https://github.com/jaloliddin-rus/local-llm-claude-code-guide" class="btn fs-5" target="_blank" rel="noopener">View on GitHub</a>
  </div>
</section>

Every number and config in this guide is measured on real hardware, not estimated. Confirmed on RTX A5000 24 GB + Linux Mint 22.3. Should transfer identically to RTX A5500 or any other 24 GB Ampere card.

---

## What you'll end up with

Three local models, each tuned for its strength:

| Model | Generation speed | Use for |
|---|---|---|
| **Qwen 3.6 27B** | ~25 tok/s | Claude Code agentic coding (it's trained for this) |
| **Gemma 4 26B-A4B (MoE)** | **~102 tok/s** | Fast daily writing, brainstorming, planning |
| **Gemma 4 31B (dense)** | ~29 tok/s | Highest-quality writing, careful analysis |

Six shell functions on the Linux box to start/switch models (`llm-code`, `llm-gemma-26b`, etc). Two shell functions on the Mac to switch between local and cloud Claude (`claude-local`, `claude-cloud`). All on one port. Swap models in ~10 seconds.

---

## The 14 steps

1. [Hardware prerequisites](./01-hardware.html)
2. [Build llama.cpp on the Linux box](./02-build-llamacpp.html)
3. [Disable ECC for extra VRAM](./03-disable-ecc.html) (optional but recommended)
4. [Download the models from Unsloth](./04-download-models.html)
5. [Shell functions on Linux](./05-shell-functions-linux.html)
6. [Verify each model loads](./06-verify-models.html)
7. [Configure SSH on the Mac](./07-configure-ssh.html)
8. [Claude Code settings on the Mac](./08-claude-code-settings.html)
9. [Shell functions on the Mac](./09-shell-functions-mac.html)
10. [End-to-end test](./10-end-to-end-test.html)
11. [Daily workflow](./11-daily-workflow.html)
12. [Reducing hallucinations on niche tools](./12-reducing-hallucinations.html)
13. [Troubleshooting](./13-troubleshooting.html)
14. [Appendix — what we actually measured](./14-appendix.html)

---

## Quick-reference card

**Linux box (SSH in first):**
```bash
llm-code              # Qwen 3.6 27B, fast preset (coding)
llm-code-think        # Qwen 3.6 27B, thinking preset
llm-gemma-26b         # Gemma 4 26B-A4B MoE (fast writing)
llm-gemma-26b-think   # Gemma 4 26B-A4B MoE, thinking
llm-gemma-31b         # Gemma 4 31B dense (best quality writing)
llm-gemma-31b-think   # Gemma 4 31B dense, thinking
llm-status            # what's running?
llm-stop              # kill it
```

**Mac:**
```bash
ssh a5000             # open tunnel — keep terminal alive
claude-local          # Claude Code → local (any running model)
claude-cloud          # Claude Code → real Anthropic
claude-status         # tunnel health
claude-update         # manual Claude Code update
```

**Daily flow:**
1. Mac terminal 1: `ssh a5000` → `llm-code` (or whichever model)
2. Mac terminal 2: `cd ~/project && claude-local` → work
3. End of day: `llm-stop` (optional)
4. Need to switch models? Go to terminal 1, `llm-gemma-26b`, back to terminal 2.
5. Need cloud? `claude-cloud` — strips local env.

---

## What you have that most people don't

- **Full privacy** — prompts and file contents never leave your LAN in local mode
- **Zero marginal cost** — unlimited agent loops, no token bills
- **Three purpose-tuned models** — coding, fast writing, best writing
- **Clean cloud escape hatch** — `claude-cloud` for when local isn't enough
- **Empirically measured config** — every number here was verified on real hardware
- **Architecturally-correct choices** — not one-size-fits-all; each model tuned to its own constraints

## When to use cloud instead

Local doesn't replace cloud Claude. Use `claude-cloud` when:
- Genuinely hard reasoning — Opus 4.7 > Qwen 27B / Gemma 31B
- Niche scientific/technical tools where hallucinations are costly
- You want the latest knowledge (cloud updates more often than your local model weights)
- Deep multi-step planning where local's ~25 tok/s feels too slow

The local setup gives you the freedom to choose the right tool per task, not replace one with the other.
