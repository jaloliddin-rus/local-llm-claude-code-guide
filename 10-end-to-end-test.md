---
title: "10. End-to-end test"
nav_order: 11
---

## Terminal 1 on Mac — SSH tunnel

```bash
ssh a5000        # your SSH config alias
# ... in the Linux shell ...
llm-code         # or llm-gemma-26b, llm-gemma-31b
# wait for: ✓ llama-server :8080  model: qwen3.6-27b
```

Leave this terminal open — it holds the tunnel alive.

## Terminal 2 on Mac — Claude Code

```bash
cd ~/some-project
claude-local
```

Claude Code's header should show:
```
Claude Code v2.x.x
qwen3.6-27b with ... effort · API Usage Billing
```

The model name in the header confirms it's using your local server, not cloud.

## Ignore this warning

> `● Prompt caching disabled via DISABLE_PROMPT_CACHING. This will impact latency and token costs.`

Claude Code assumes you're on cloud (where disabling caching is bad). For local it's required — llama.cpp doesn't implement Anthropic's caching protocol. Ignore.

## Measure actual speed from the Linux side

In the SSH terminal:

```bash
tail -f ~/.llama-server.log | grep -E "eval time"
```

Then send a Claude Code message. You'll see two timing lines per turn — prompt eval (input processing) and eval (generation).

Healthy numbers on 24 GB Ampere:

| Model | Prompt eval | Generation |
|---|---|---|
| Qwen 3.6 27B | 600-900 tok/s | ~25 tok/s |
| Gemma 4 26B-A4B | 600-900 tok/s | **~100 tok/s** |
| Gemma 4 31B | 150-300 tok/s | ~29 tok/s |

---

Next: [11. Daily workflow](./11-daily-workflow.html)
