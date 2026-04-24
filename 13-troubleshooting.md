---
title: "13. Troubleshooting"
nav_order: 14
---

## Claude Code

| Symptom | Fix |
|---|---|
| `claude-local` says "No tunnel" | SSH session not active, or started before config change. `exit` and run `ssh a5000` again. |
| Slow/repetitive outputs after turn 2 | `CLAUDE_CODE_ATTRIBUTION_HEADER=0` not in `~/.claude/settings.json` (the shell-export version doesn't work, must be in file) |
| Tool calls silently fail | Server missing `--jinja` flag |
| Still prompting login | `echo $ANTHROPIC_BASE_URL` inside `claude-local` — should be set. If not, function isn't loading. |
| Prompt caching warning | Ignore — required for local llama.cpp |

## llama-server

| Symptom | Fix |
|---|---|
| "✗ not running" after starting | `sleep 12` in function not long enough for first cold load (NVMe cache warmup). Re-run the function; second load is fast. |
| Out of memory on load | ECC still enabled? Check with `nvidia-smi -q -d ECC`. If that's not it, lower `--ctx-size`. |
| `CPU KV buffer size` in log | VRAM too tight. For Gemma 26B: that shouldn't happen; check you removed `--cache-type-k q8_0`. For 31B: expected at higher ctx, drop `--ctx-size`. |
| Gibberish output | CUDA 13.2? Check with `nvidia-smi`. Downgrade to 13.1 or 12.x. |
| Generation very slow (<15 tok/s) | Model fell back to CPU. Check `nvidia-smi` shows VRAM used by llama-server. Rebuild llama.cpp with CUDA. |
| `setsockopt TCP_NODELAY: Invalid argument` | Cosmetic log noise. Known harmless bug. Ignore. |
| Model not found error | Path in `~/.bashrc` doesn't match where you downloaded. Check `ls ~/.lmstudio/models/unsloth/`. |

## Network / SSH

| Symptom | Fix |
|---|---|
| SSH drops after Mac sleeps | Reconnect: `ssh a5000`. `ServerAliveInterval 60` helps but isn't bulletproof. |
| Port 8080 already in use on Mac | Previous SSH session holds it. `pkill -f "ssh a5000"` on Mac, retry. |

## LM Studio gotchas (if you also use it)

| Symptom | Fix |
|---|---|
| `lms get` downloads wrong quant | LM Studio's picker maps your selection to their curated mirror, not Unsloth. **Use `hf download` directly** as in [step 4](./04-download-models.html) of this guide. |
| `lms ls` shows ghost models that don't exist on disk | LM Studio's `model-data.json` is stale. With LM Studio service stopped (`pkill -f llmster`), overwrite: `echo '{"json":[],"meta":{"values":["map"]}}' > ~/.lmstudio/.internal/model-data.json`. Next `lms ls` rebuilds from filesystem. |
| `lms ls` shows the bundled `nomic` embedding you never downloaded | It's a bundled model shipped with LM Studio. Harmless. |

---

Next: [14. Appendix — what we actually measured](./14-appendix.html)
