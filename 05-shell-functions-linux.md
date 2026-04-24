---
title: "5. Shell functions on Linux"
nav_order: 6
---

Add all of this to `~/.bashrc`. Six functions total plus helpers.

**Before you paste:** adjust the `MODELS_BASE` variable if you put models somewhere else.

```bash
cat >> ~/.bashrc <<'EOF'

# ============================================================
# Local LLM server (llama-server) — Qwen 3.6 + Gemma 4
# ============================================================
MODELS_BASE="$HOME/.lmstudio/models"
LLAMA_PORT=8080
LLAMA_LOG="$HOME/.llama-server.log"

# Model paths
QWEN_MODEL="$MODELS_BASE/unsloth/Qwen3.6-27B-GGUF/Qwen3.6-27B-UD-Q4_K_XL.gguf"
GEMMA_26B_MODEL="$MODELS_BASE/unsloth/gemma-4-26B-A4B-it-GGUF/gemma-4-26B-A4B-it-UD-Q4_K_XL.gguf"
GEMMA_26B_MMPROJ="$MODELS_BASE/unsloth/gemma-4-26B-A4B-it-GGUF/mmproj-BF16.gguf"
GEMMA_31B_MODEL="$MODELS_BASE/unsloth/gemma-4-31B-it-GGUF/gemma-4-31B-it-UD-Q4_K_XL.gguf"
GEMMA_31B_MMPROJ="$MODELS_BASE/unsloth/gemma-4-31B-it-GGUF/mmproj-BF16.gguf"

# ---- Helpers ---------------------------------------------------
llm-status() {
    if curl -sf "http://localhost:${LLAMA_PORT}/health" >/dev/null 2>&1; then
        local name=$(curl -s "http://localhost:${LLAMA_PORT}/v1/models" \
            | grep -o '"id":"[^"]*"' | head -1 | cut -d'"' -f4)
        echo "✓ llama-server :${LLAMA_PORT}  model: ${name}"
        return 0
    else
        echo "✗ llama-server not running"
        return 1
    fi
}

llm-stop() {
    pkill -f "llama-server.*--port ${LLAMA_PORT}" 2>/dev/null
    sleep 1
    echo "stopped."
}

# ---- Qwen 3.6 27B (coding) -----------------------------------
# Daily driver for Claude Code.
# Unsloth non-thinking "general tasks" preset.
# q8_0 KV needed — Qwen's bigger KV cache would force CPU spillover at f16.
llm-code() {
    llm-stop
    if [[ ! -f "$QWEN_MODEL" ]]; then
        echo "Model not found: $QWEN_MODEL"
        return 1
    fi
    echo "Starting Qwen 3.6 27B (non-thinking, fast preset)..."
    nohup llama-server \
        -m "$QWEN_MODEL" \
        --alias qwen3.6-27b \
        --port ${LLAMA_PORT} --host 127.0.0.1 \
        --jinja \
        --ctx-size 131072 \
        --flash-attn on \
        --cache-type-k q8_0 --cache-type-v q8_0 \
        --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.0 \
        --presence-penalty 1.5 \
        --chat-template-kwargs '{"enable_thinking":false}' \
        > "$LLAMA_LOG" 2>&1 &
    sleep 12
    llm-status
}

# Qwen thinking mode — for hard debugging only. Slow.
# Unsloth "precise coding" preset.
llm-code-think() {
    llm-stop
    if [[ ! -f "$QWEN_MODEL" ]]; then
        echo "Model not found: $QWEN_MODEL"
        return 1
    fi
    echo "Starting Qwen 3.6 27B (thinking mode, precise coding)..."
    nohup llama-server \
        -m "$QWEN_MODEL" \
        --alias qwen3.6-27b-think \
        --port ${LLAMA_PORT} --host 127.0.0.1 \
        --jinja \
        --ctx-size 131072 \
        --flash-attn on \
        --cache-type-k q8_0 --cache-type-v q8_0 \
        --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 \
        --presence-penalty 0.0 \
        --chat-template-kwargs '{"enable_thinking":true}' \
        > "$LLAMA_LOG" 2>&1 &
    sleep 12
    llm-status
}

# ---- Gemma 4 26B-A4B (fast MoE writing/planning) --------------
# ~100 tok/s generation — fastest model on this setup.
# f16 KV cache (Unsloth default) — Gemma's hybrid attention means
# small KV cache even at 64K, so no need to quantize.
# Includes vision support via mmproj.
llm-gemma-26b() {
    llm-stop
    if [[ ! -f "$GEMMA_26B_MODEL" ]]; then
        echo "Model not found: $GEMMA_26B_MODEL"
        return 1
    fi
    echo "Starting Gemma 4 26B-A4B (thinking off, 64K ctx)..."
    local mmproj_flag=""
    [[ -f "$GEMMA_26B_MMPROJ" ]] && mmproj_flag="--mmproj $GEMMA_26B_MMPROJ"
    nohup llama-server \
        -m "$GEMMA_26B_MODEL" $mmproj_flag \
        --alias gemma-4-26b-a4b \
        --port ${LLAMA_PORT} --host 127.0.0.1 \
        --jinja \
        --ctx-size 65536 \
        --flash-attn on \
        --temp 1.0 --top-p 0.95 --top-k 64 \
        --chat-template-kwargs '{"enable_thinking":false}' \
        > "$LLAMA_LOG" 2>&1 &
    sleep 12
    llm-status
}

llm-gemma-26b-think() {
    llm-stop
    if [[ ! -f "$GEMMA_26B_MODEL" ]]; then
        echo "Model not found: $GEMMA_26B_MODEL"
        return 1
    fi
    echo "Starting Gemma 4 26B-A4B (thinking ON, 64K ctx)..."
    local mmproj_flag=""
    [[ -f "$GEMMA_26B_MMPROJ" ]] && mmproj_flag="--mmproj $GEMMA_26B_MMPROJ"
    nohup llama-server \
        -m "$GEMMA_26B_MODEL" $mmproj_flag \
        --alias gemma-4-26b-a4b-think \
        --port ${LLAMA_PORT} --host 127.0.0.1 \
        --jinja \
        --ctx-size 65536 \
        --flash-attn on \
        --temp 1.0 --top-p 0.95 --top-k 64 \
        --chat-template-kwargs '{"enable_thinking":true}' \
        > "$LLAMA_LOG" 2>&1 &
    sleep 12
    llm-status
}

# ---- Gemma 4 31B (highest-quality dense) ----------------------
# Slower than 26B-A4B (~29 tok/s vs ~100) but measurably better prose.
# KEEPS q8_0 KV cache — 31B is VRAM-tight on 24 GB, f16 would OOM.
# Lower 32K ctx for the same reason.
llm-gemma-31b() {
    llm-stop
    if [[ ! -f "$GEMMA_31B_MODEL" ]]; then
        echo "Model not found: $GEMMA_31B_MODEL"
        return 1
    fi
    echo "Starting Gemma 4 31B (thinking off, 32K ctx)..."
    local mmproj_flag=""
    [[ -f "$GEMMA_31B_MMPROJ" ]] && mmproj_flag="--mmproj $GEMMA_31B_MMPROJ"
    nohup llama-server \
        -m "$GEMMA_31B_MODEL" $mmproj_flag \
        --alias gemma-4-31b \
        --port ${LLAMA_PORT} --host 127.0.0.1 \
        --jinja \
        --ctx-size 32768 \
        --flash-attn on \
        --cache-type-k q8_0 --cache-type-v q8_0 \
        --temp 1.0 --top-p 0.95 --top-k 64 \
        --chat-template-kwargs '{"enable_thinking":false}' \
        > "$LLAMA_LOG" 2>&1 &
    sleep 12
    llm-status
}

llm-gemma-31b-think() {
    llm-stop
    if [[ ! -f "$GEMMA_31B_MODEL" ]]; then
        echo "Model not found: $GEMMA_31B_MODEL"
        return 1
    fi
    echo "Starting Gemma 4 31B (thinking ON, 32K ctx)..."
    local mmproj_flag=""
    [[ -f "$GEMMA_31B_MMPROJ" ]] && mmproj_flag="--mmproj $GEMMA_31B_MMPROJ"
    nohup llama-server \
        -m "$GEMMA_31B_MODEL" $mmproj_flag \
        --alias gemma-4-31b-think \
        --port ${LLAMA_PORT} --host 127.0.0.1 \
        --jinja \
        --ctx-size 32768 \
        --flash-attn on \
        --cache-type-k q8_0 --cache-type-v q8_0 \
        --temp 1.0 --top-p 0.95 --top-k 64 \
        --chat-template-kwargs '{"enable_thinking":true}' \
        > "$LLAMA_LOG" 2>&1 &
    sleep 12
    llm-status
}
EOF

source ~/.bashrc
```

## Key design decisions per model

| Model | KV cache | Context | Sampling | Reason |
|---|---|---|---|---|
| **Qwen 3.6 27B** | q8_0 | 128K | temp=0.7, top_p=0.8, top_k=20, presence=1.5 | Unsloth non-thinking general preset; q8_0 forced by bigger KV footprint; presence penalty prevents agent-loop repetition |
| **Gemma 4 26B-A4B** | f16 (default) | 64K | temp=1.0, top_p=0.95, top_k=64 | Google's recommended Gemma defaults; MoE+hybrid = tiny KV so f16 fits easily; no presence penalty (hurts Gemma) |
| **Gemma 4 31B** | q8_0 | 32K | temp=1.0, top_p=0.95, top_k=64 | Same sampling as 26B; q8_0 required because 31B dense is VRAM-tight at 24 GB |

These aren't arbitrary — each was verified to fit all-GPU on 24 GB Ampere with real measurements.

## Verify the functions loaded

```bash
type llm-code
type llm-gemma-26b
type llm-gemma-31b
```

All should print function bodies.

---

Next: [6. Verify each model loads](./06-verify-models.html)
