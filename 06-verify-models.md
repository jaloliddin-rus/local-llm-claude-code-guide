---
title: "6. Verify each model loads"
nav_order: 7
---

Test each in sequence. Each test confirms: loads without OOM, KV cache stays on GPU, produces real text.

## Qwen 3.6 27B

```bash
llm-code
nvidia-smi --query-gpu=memory.used,memory.free --format=csv
grep -E "K \(|V \(|CPU KV buffer" ~/.llama-server.log | tail -5
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3.6-27b","messages":[{"role":"user","content":"Hello in 5 words."}]}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

Expected numbers on 24 GB (ECC off):
- **~21.9 GB used, ~2.1 GB free**
- **No `CPU KV buffer` line**
- **Clean response text**

## Gemma 4 26B-A4B

```bash
llm-gemma-26b
nvidia-smi --query-gpu=memory.used,memory.free --format=csv
grep -E "K \(|V \(|CPU KV buffer" ~/.llama-server.log | tail -5
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gemma-4-26b-a4b","messages":[{"role":"user","content":"Say hi in 5 words."}]}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

Expected:
- **~20.6 GB used, ~3.4 GB free**
- **`K (f16)`** in log, no CPU spillover
- Fast response (generation ~100 tok/s)

## Gemma 4 31B

```bash
llm-gemma-31b
nvidia-smi --query-gpu=memory.used,memory.free --format=csv
grep -E "K \(|V \(|CPU KV buffer" ~/.llama-server.log | tail -5
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gemma-4-31b","messages":[{"role":"user","content":"Hello in 5 words."}]}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

Expected:
- **~23.4 GB used, ~0.7 GB free** (tight — intentional, to make 31B fit)
- **`K (q8_0)`** in log, no CPU spillover
- Response around 30 tok/s

If any test fails with OOM, check ECC is actually disabled. If `CPU KV buffer` appears in the log, drop `--ctx-size` in that function.

## Confirm the Anthropic endpoint works

This is the one Claude Code actually uses:

```bash
curl -s http://localhost:8080/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"gemma-4-31b","max_tokens":50,"messages":[{"role":"user","content":"Hello."}]}' \
  | python3 -m json.tool
```

Response should have `"type": "message"` and `content` as an array — that's Anthropic's format, which means llama.cpp is properly translating. If this fails, your llama.cpp build is older than b4847 — rebuild from latest `main`.

---

Next: [7. Configure SSH on the Mac](./07-configure-ssh.html)
