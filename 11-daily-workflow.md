---
title: "11. Daily workflow"
nav_order: 12
---

## Start of day

```bash
# Mac terminal 1 — tunnel
ssh a5000
llm-code              # or whichever model you want
# leave this terminal open

# Mac terminal 2 — work
cd ~/my-project
claude-local
```

## Switching models

From inside the SSH session:

```bash
llm-stop              # kill current
llm-code              # Qwen for coding
llm-gemma-26b         # Gemma MoE for writing
llm-gemma-31b         # Gemma dense for deep analysis
llm-gemma-26b-think   # any-model-think variant
```

Swap takes ~10 seconds. Claude Code on the Mac doesn't care which is running — `claude-local` auto-detects.

## Switching providers (local vs cloud)

On the Mac:
```bash
claude-local          # whatever's running on port 8080
claude-cloud          # real Anthropic — strips local env
```

Use `claude-cloud` when:
- Working on niche technical domains (cloud hallucinates less)
- Deep reasoning problems where Opus 4.7 > local 27B
- Local isn't started and you don't want to start it

Use `claude-local` when:
- Privacy matters (prompts never leave your LAN)
- Unmetered iteration
- General daily coding

## End of day (optional)

```bash
# From Mac, one-liner — kills the running model:
ssh a5000 'source ~/.bashrc && llm-stop'
```

Or just leave the server running — it holds VRAM but zero CPU when idle.

## Updating Claude Code

Auto-updater is off, so monthly:
```bash
claude-update
```

---

Next: [12. Reducing hallucinations on niche tools](./12-reducing-hallucinations.html)
