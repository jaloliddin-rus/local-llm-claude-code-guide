---
title: Home
layout: default
nav_order: 1
---

<section class="hero">
  <h1>Local LLM + Claude Code</h1>
  <p class="hero-subtitle">Run Qwen 3.6 and Gemma 4 locally on a 24&nbsp;GB GPU. Talk to them from Claude Code over SSH. Full privacy, zero marginal cost.</p>
  <div class="hero-buttons">
    <a href="#1-hardware-prerequisites" class="btn btn-purple fs-5">Get started →</a>
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

## Table of contents

1. [Hardware prerequisites](#1-hardware-prerequisites)
2. [Build llama.cpp on the Linux box](#2-build-llamacpp-on-the-linux-box)
3. [Disable ECC for extra VRAM (optional but recommended)](#3-disable-ecc-for-extra-vram-optional-but-recommended)
4. [Download the models from Unsloth](#4-download-the-models-from-unsloth)
5. [Write the llm-* shell functions on Linux](#5-write-the-llm--shell-functions-on-linux)
6. [Verify each model loads](#6-verify-each-model-loads)
7. [Configure SSH on the Mac](#7-configure-ssh-on-the-mac)
8. [Claude Code settings on the Mac](#8-claude-code-settings-on-the-mac)
9. [Shell functions on the Mac](#9-shell-functions-on-the-mac)
10. [End-to-end test](#10-end-to-end-test)
11. [Daily workflow](#11-daily-workflow)
12. [Reducing hallucinations on niche tools](#12-reducing-hallucinations-on-niche-tools)
13. [Troubleshooting](#13-troubleshooting)
14. [Appendix — what we actually measured](#14-appendix--what-we-actually-measured)

---

## 1. Hardware prerequisites

### On the Linux box (where models run)

```bash
nvidia-smi
```

You need:
- **NVIDIA GPU with 24 GB VRAM** (RTX A5000, A5500, 3090, 4090, etc.)
- **NVIDIA driver 535+** for recent CUDA support
- **CUDA version NOT 13.2** — Unsloth reports gibberish outputs on CUDA 13.2 for both Qwen 3.6 and Gemma 4. Check with `nvidia-smi | grep CUDA`. 12.x, 13.0, or 13.1 are fine.

```bash
# Check disk space (we'll need ~60 GB for models)
df -h ~
```

Put models on your fastest disk. If you have both NVMe and HDD, use NVMe — model loads are 15-20× faster.

### On the Mac (where Claude Code runs)

```bash
claude --version
```

If missing:
```bash
npm install -g @anthropic-ai/claude-code
# if npm is missing: brew install node
```

---

## 2. Build llama.cpp on the Linux box

We use native llama.cpp (not LM Studio's embedded version) because:

- It speaks the Anthropic Messages API natively at `/v1/messages` (since b4847, Jan 2026) — so Claude Code connects with no proxy
- It supports `--chat-template-kwargs` for toggling thinking mode on Qwen and Gemma
- Gets updates the day llama.cpp publishes them

### Install build tools

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake git libcurl4-openssl-dev
```

### Clone and build with CUDA

```bash
cd ~
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

cmake -B build -DGGML_CUDA=ON -DBUILD_SHARED_LIBS=OFF
cmake --build build --config Release -j --target llama-server llama-cli llama-gguf-split llama-mtmd-cli
```

Takes 5-15 minutes. The `llama-mtmd-cli` target is for multimodal (vision) support, which Gemma 4 needs.

### Install binaries on PATH

```bash
sudo install build/bin/llama-* /usr/local/bin/
```

### Verify

```bash
cd ~
llama-server --version
```

You should see output like:
```
ggml_cuda_init: found 1 CUDA devices (Total VRAM: 24564 MiB):
  Device 0: NVIDIA RTX A5000, compute capability 8.6, VMM: yes, VRAM: 24564 MiB
version: 8753 (3fc65063d)
```

If you see `found 0 CUDA devices` or no CUDA line at all, the build didn't link CUDA. Install `nvidia-cuda-toolkit` and rebuild.

---

## 3. Disable ECC for extra VRAM (optional but recommended)

Workstation cards ship with ECC on, reserving ~1.5 GB. For LLM inference, ECC is overkill — the risk of a cosmic-ray bit flip during a single turn is vanishingly small, and even if it happened you'd just see one weird token. Disabling ECC gets you the full 24 GB instead of ~22.5 GB.

### Check current state

```bash
nvidia-smi -q -d ECC | grep "Current"
```

If it says `Enabled`:

```bash
sudo nvidia-smi -e 0
sudo reboot
```

After reboot:
```bash
nvidia-smi -q -d ECC | grep "Current"    # should say Disabled
nvidia-smi --query-gpu=memory.total --format=csv
# Should now report ~24564 MiB (up from ~23028)
```

**Why this matters:** the extra 1.5 GB is what lets Qwen 3.6 run cleanly at 128K context. Without it, you'd cap Qwen at 64K.

To re-enable later: `sudo nvidia-smi -e 1 && sudo reboot`.

---

## 4. Download the models from Unsloth

**Don't use LM Studio's `lms get` for this.** Its picker defaults to mirrors like `lmstudio-community` which don't have Unsloth's Dynamic 2.0 quants. Go straight to HuggingFace.

### Install `hf` CLI

```bash
pip install --user huggingface_hub hf_transfer
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
hf --version
```

If `pip` errors with "externally-managed-environment" on newer Ubuntu:
```bash
python3 -m venv ~/.hf_venv
~/.hf_venv/bin/pip install huggingface_hub hf_transfer
echo 'export PATH="$HOME/.hf_venv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Pick a location

Models should live on your NVMe. If you're using LM Studio too, put them under its models directory so LM Studio can see them:

```bash
MODELS_BASE="$HOME/.lmstudio/models"   # or wherever
mkdir -p "$MODELS_BASE"
```

If you're not using LM Studio, any path works. Just remember where you put them for the shell functions in step 5.

### Download the three models

Each command downloads only the UD-Q4_K_XL quant plus the multimodal projector (for vision), skipping all other quants in the repo.

**Qwen 3.6 27B** (coding, ~17 GB):
```bash
HF_HUB_ENABLE_HF_TRANSFER=1 hf download unsloth/Qwen3.6-27B-GGUF \
    --local-dir "$MODELS_BASE/unsloth/Qwen3.6-27B-GGUF" \
    --include "*UD-Q4_K_XL*" \
    --include "*mmproj-F32*"
```

**Gemma 4 26B-A4B** (fast writing MoE, ~17 GB):
```bash
HF_HUB_ENABLE_HF_TRANSFER=1 hf download unsloth/gemma-4-26B-A4B-it-GGUF \
    --local-dir "$MODELS_BASE/unsloth/gemma-4-26B-A4B-it-GGUF" \
    --include "*UD-Q4_K_XL*" \
    --include "*mmproj-BF16*"
```

**Gemma 4 31B** (highest-quality dense, ~21 GB):
```bash
HF_HUB_ENABLE_HF_TRANSFER=1 hf download unsloth/gemma-4-31B-it-GGUF \
    --local-dir "$MODELS_BASE/unsloth/gemma-4-31B-it-GGUF" \
    --include "*UD-Q4_K_XL*" \
    --include "*mmproj-BF16*"
```

Verify all three landed:

```bash
find "$MODELS_BASE/unsloth" -name "*.gguf" -exec ls -lh {} \;
```

You should see 6 files total — three main GGUFs (17-21 GB each) and three mmproj files (1-2 GB each).

Total disk: about 58 GB.

---

## 5. Write the llm-* shell functions on Linux

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

### Key design decisions per model

| Model | KV cache | Context | Sampling | Reason |
|---|---|---|---|---|
| **Qwen 3.6 27B** | q8_0 | 128K | temp=0.7, top_p=0.8, top_k=20, presence=1.5 | Unsloth non-thinking general preset; q8_0 forced by bigger KV footprint; presence penalty prevents agent-loop repetition |
| **Gemma 4 26B-A4B** | f16 (default) | 64K | temp=1.0, top_p=0.95, top_k=64 | Google's recommended Gemma defaults; MoE+hybrid = tiny KV so f16 fits easily; no presence penalty (hurts Gemma) |
| **Gemma 4 31B** | q8_0 | 32K | temp=1.0, top_p=0.95, top_k=64 | Same sampling as 26B; q8_0 required because 31B dense is VRAM-tight at 24 GB |

These aren't arbitrary — each was verified to fit all-GPU on 24 GB Ampere with real measurements.

### Verify the functions loaded

```bash
type llm-code
type llm-gemma-26b
type llm-gemma-31b
```

All should print function bodies.

---

## 6. Verify each model loads

Test each in sequence. Each test confirms: loads without OOM, KV cache stays on GPU, produces real text.

### Qwen 3.6 27B

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

### Gemma 4 26B-A4B

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

### Gemma 4 31B

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

### Confirm the Anthropic endpoint works

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

## 7. Configure SSH on the Mac

Now set up an SSH tunnel so the Mac reaches port 8080 on the Linux box.

### Edit `~/.ssh/config` on the Mac

```bash
nano ~/.ssh/config
```

Add (replace `<LINUX_IP>` and `<USERNAME>`):

```
Host a5000
    Hostname <LINUX_IP>
    User <USERNAME>
    LocalForward 8080 localhost:8080
    LocalForward 1234 localhost:1234
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Save (Ctrl+O, Enter, Ctrl+X).

The two port forwards:
- **8080** — llama-server. Claude Code hits `localhost:8080` on the Mac, SSH forwards it to the Linux box.
- **1234** — LM Studio's GUI (if you use it). Visit `http://localhost:1234` in your Mac browser after SSH'ing to use LM Studio's chat UI.

For your A5500 box later, just add another entry:
```
Host a5500
    Hostname <A5500_IP>
    User <USERNAME>
    LocalForward 8080 localhost:8080
    ...
```

Note: only one tunnel at a time can hold Mac's port 8080, so you'd pick one box per work session.

Verify the config parsed:
```bash
ssh -G a5000 | grep -E "^hostname|^user |localforward"
```

Expected: one hostname, one user, two localforward lines.

---

## 8. Claude Code settings on the Mac

Create `~/.claude/settings.json` with privacy-hardened defaults that are safe for both local and cloud use.

```bash
mkdir -p ~/.claude
nano ~/.claude/settings.json
```

If the file is empty, paste:

```json
{
  "env": {
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "DISABLE_TELEMETRY": "1",
    "DISABLE_ERROR_REPORTING": "1",
    "DISABLE_BUG_COMMAND": "1",
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

If the file has other keys (plugin configs, etc.), add the `"env"` object alongside them (don't forget the comma before it).

Verify the JSON is valid:
```bash
python3 -m json.tool ~/.claude/settings.json
```

### What each setting does

| Variable | Effect | Safe for cloud? |
|---|---|---|
| `CLAUDE_CODE_ATTRIBUTION_HEADER=0` | Skips metadata header — **critical for local** (otherwise KV cache invalidates every turn and inference drops 5-10×) | ✅ harmless on cloud |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` | Enterprise-grade telemetry block | ✅ |
| `DISABLE_TELEMETRY=1` | OpenTelemetry metrics off | ✅ |
| `DISABLE_ERROR_REPORTING=1` | Sentry crash reports off | ✅ |
| `DISABLE_BUG_COMMAND=1` | Disables `/bug` slash command | ✅ |
| `DISABLE_AUTOUPDATER=1` | Stop auto-updater; use `claude-update` manually | ✅ |

**`DISABLE_PROMPT_CACHING=1`** is also required for local but goes in the shell function, not here — it costs 90% on cloud. See step 9.

---

## 9. Shell functions on the Mac

Add to `~/.zshrc` (or `~/.bashrc` for bash):

```bash
nano ~/.zshrc
```

Paste at the bottom:

```bash
# ============================================================
# Claude Code — local (via tunnel) vs cloud
# ============================================================

claude-local() {
    if ! curl -sf http://localhost:8080/v1/models >/dev/null 2>&1; then
        echo "✗ No tunnel. In another terminal, run: ssh a5000"
        return 1
    fi
    # Detect which model is running from /v1/models
    local model=$(curl -s http://localhost:8080/v1/models \
        | python3 -c "import sys,json; print(json.load(sys.stdin)['data'][0]['id'])" 2>/dev/null)
    ANTHROPIC_BASE_URL="http://localhost:8080" \
    ANTHROPIC_AUTH_TOKEN="sk-local" \
    ANTHROPIC_MODEL="$model" \
    DISABLE_PROMPT_CACHING=1 \
    command claude "$@"
}

claude-cloud() {
    env -u ANTHROPIC_BASE_URL -u ANTHROPIC_AUTH_TOKEN -u ANTHROPIC_MODEL \
        -u DISABLE_PROMPT_CACHING \
        command claude "$@"
}

claude-status() {
    if curl -sf http://localhost:8080/v1/models >/dev/null 2>&1; then
        local name=$(curl -s http://localhost:8080/v1/models \
            | python3 -c "import sys,json; print(json.load(sys.stdin)['data'][0]['id'])" 2>/dev/null)
        echo "✓ local tunnel up — model: $name"
    else
        echo "✗ local tunnel down — run: ssh a5000 (in another terminal)"
    fi
}

# Manual Claude Code update (auto-updater disabled in settings.json)
alias claude-update='npm update -g @anthropic-ai/claude-code && claude --version'
```

Reload:
```bash
source ~/.zshrc
```

Verify:
```bash
type claude-local
type claude-cloud
claude-status    # should say "✓ local tunnel up" if ssh a5000 is active
```

### The critical design choice

`claude-local` **auto-detects the running model** by querying `/v1/models` and sets `ANTHROPIC_MODEL` accordingly. This means one function works whether you're running Qwen, Gemma 26B, or Gemma 31B — you just start a different model on the Linux side and `claude-local` picks it up on the next run.

`claude-cloud` uses `env -u` to **strip** local-only env vars, so cloud calls never get confused.

---

## 10. End-to-end test

### Terminal 1 on Mac — SSH tunnel

```bash
ssh a5000        # your SSH config alias
# ... in the Linux shell ...
llm-code         # or llm-gemma-26b, llm-gemma-31b
# wait for: ✓ llama-server :8080  model: qwen3.6-27b
```

Leave this terminal open — it holds the tunnel alive.

### Terminal 2 on Mac — Claude Code

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

### Ignore this warning

> `● Prompt caching disabled via DISABLE_PROMPT_CACHING. This will impact latency and token costs.`

Claude Code assumes you're on cloud (where disabling caching is bad). For local it's required — llama.cpp doesn't implement Anthropic's caching protocol. Ignore.

### Measure actual speed from the Linux side

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

## 11. Daily workflow

### Start of day

```bash
# Mac terminal 1 — tunnel
ssh a5000
llm-code              # or whichever model you want
# leave this terminal open

# Mac terminal 2 — work
cd ~/my-project
claude-local
```

### Switching models

From inside the SSH session:

```bash
llm-stop              # kill current
llm-code              # Qwen for coding
llm-gemma-26b         # Gemma MoE for writing
llm-gemma-31b         # Gemma dense for deep analysis
llm-gemma-26b-think   # any-model-think variant
```

Swap takes ~10 seconds. Claude Code on the Mac doesn't care which is running — `claude-local` auto-detects.

### Switching providers (local vs cloud)

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

### End of day (optional)

```bash
# From Mac, one-liner — kills the running model:
ssh a5000 'source ~/.bashrc && llm-stop'
```

Or just leave the server running — it holds VRAM but zero CPU when idle.

### Updating Claude Code

Auto-updater is off, so monthly:
```bash
claude-update
```

---

## 12. Reducing hallucinations on niche tools

Open 27B/31B models have fuzzy recall on niche APIs — specialized command flags, scientific tool options, library internals. Both Qwen and Gemma will confidently invent function names on specialty domains.

**The mitigation that actually works:** give the model a reference file in your project and tell it to `grep` before writing commands. Claude Code will follow this instruction because it has a Bash tool.

### Pattern: project-level CLAUDE.md

In a project folder:

```bash
cd ~/projects/my-niche-work

# Scrape help output from the tool
mkdir -p help
for cmd in <list_of_commands>; do
    "$cmd" -help > help/$cmd.txt 2>&1
done
```

Create `CLAUDE.md` in the same directory:

```markdown
# Tool reference

Before writing any command for <niche tool>, grep the help files:

    grep -B 1 -A 3 "<flag_name>" ./help/<command>.txt

If the flag isn't in the help file, do NOT invent one. Say:
"I don't see this flag documented — please check `<command> -help`."

## Known hallucinations to avoid

- `<specific wrong thing you've caught>` — correct form is `<right thing>`
- ...
```

Claude Code auto-loads `CLAUDE.md` from the current directory. The model sees these instructions and follows them (Qwen and Gemma both comply ~85-95% for well-structured rules).

**When you catch a new hallucination, add it to the "Known hallucinations" section.** Over time this becomes a robust blacklist for that tool.

This doesn't eliminate hallucinations — it reduces them dramatically and gives you a mechanism to keep improving.

---

## 13. Troubleshooting

### Claude Code

| Symptom | Fix |
|---|---|
| `claude-local` says "No tunnel" | SSH session not active, or started before config change. `exit` and run `ssh a5000` again. |
| Slow/repetitive outputs after turn 2 | `CLAUDE_CODE_ATTRIBUTION_HEADER=0` not in `~/.claude/settings.json` (the shell-export version doesn't work, must be in file) |
| Tool calls silently fail | Server missing `--jinja` flag |
| Still prompting login | `echo $ANTHROPIC_BASE_URL` inside `claude-local` — should be set. If not, function isn't loading. |
| Prompt caching warning | Ignore — required for local llama.cpp |

### llama-server

| Symptom | Fix |
|---|---|
| "✗ not running" after starting | `sleep 12` in function not long enough for first cold load (NVMe cache warmup). Re-run the function; second load is fast. |
| Out of memory on load | ECC still enabled? Check with `nvidia-smi -q -d ECC`. If that's not it, lower `--ctx-size`. |
| `CPU KV buffer size` in log | VRAM too tight. For Gemma 26B: that shouldn't happen; check you removed `--cache-type-k q8_0`. For 31B: expected at higher ctx, drop `--ctx-size`. |
| Gibberish output | CUDA 13.2? Check with `nvidia-smi`. Downgrade to 13.1 or 12.x. |
| Generation very slow (<15 tok/s) | Model fell back to CPU. Check `nvidia-smi` shows VRAM used by llama-server. Rebuild llama.cpp with CUDA. |
| `setsockopt TCP_NODELAY: Invalid argument` | Cosmetic log noise. Known harmless bug. Ignore. |
| Model not found error | Path in `~/.bashrc` doesn't match where you downloaded. Check `ls ~/.lmstudio/models/unsloth/`. |

### Network / SSH

| Symptom | Fix |
|---|---|
| SSH drops after Mac sleeps | Reconnect: `ssh a5000`. `ServerAliveInterval 60` helps but isn't bulletproof. |
| Port 8080 already in use on Mac | Previous SSH session holds it. `pkill -f "ssh a5000"` on Mac, retry. |

### LM Studio gotchas (if you also use it)

| Symptom | Fix |
|---|---|
| `lms get` downloads wrong quant | LM Studio's picker maps your selection to their curated mirror, not Unsloth. **Use `hf download` directly** as in step 4 of this guide. |
| `lms ls` shows ghost models that don't exist on disk | LM Studio's `model-data.json` is stale. With LM Studio service stopped (`pkill -f llmster`), overwrite: `echo '{"json":[],"meta":{"values":["map"]}}' > ~/.lmstudio/.internal/model-data.json`. Next `lms ls` rebuilds from filesystem. |
| `lms ls` shows the bundled `nomic` embedding you never downloaded | It's a bundled model shipped with LM Studio. Harmless. |

---

## 14. Appendix — what we actually measured

The numbers in this guide aren't estimates. Every one was measured on a real RTX A5000 24 GB + Linux Mint 22.3 + ECC disabled. Should transfer cleanly to RTX A5500 (same Ampere architecture, same VRAM) and similar cards.

### Qwen 3.6 27B UD-Q4_K_XL at 128K ctx, q8_0 KV

```
Model size on disk:     17 GB
Model weights in VRAM:  ~19.5 GB
KV cache (128K ctx):    4.35 GB
  K (q8_0):               2.18 GB
  V (q8_0):               2.18 GB
Total VRAM used:        21.9 GB / 24 GB
VRAM free:              ~2.1 GB
Architecture:           hybrid attention — only 16 of 64 layers use full attn
Prompt eval:            ~850 tok/s warm
Generation:             ~25 tok/s
```

### Gemma 4 26B-A4B UD-Q4_K_XL at 64K ctx, f16 KV

```
Model size on disk:     17 GB
Model weights in VRAM:  ~16 GB
KV cache (64K ctx):     2.18 GB
  Full-attn (5 layers):   1.28 GB at 65536 cells
  Sliding window (25):    0.90 GB at 4608 cells each
Total VRAM used:        20.6 GB / 24 GB
VRAM free:              ~3.4 GB
Architecture:           hybrid + MoE (4B active params)
Prompt eval:            ~630 tok/s
Generation:             **~102 tok/s**  (MoE benefit!)
```

### Gemma 4 31B UD-Q4_K_XL at 32K ctx, q8_0 KV

```
Model size on disk:     21 GB
Model weights in VRAM:  ~20 GB
KV cache (32K ctx):     3.27 GB
  Full-attn (10 layers):  1.36 GB at 32768 cells
  Sliding window (50):    1.91 GB at 4608 cells each
Total VRAM used:        23.4 GB / 24 GB
VRAM free:              ~0.7 GB  (TIGHT — intentional)
Architecture:           hybrid dense (60 attn layers)
Prompt eval:            ~160 tok/s
Generation:             ~29 tok/s
```

### Why the three configs differ

**Qwen** has larger per-layer KV footprint than Gemma at equivalent context. Had to use q8_0 KV to fit 128K. Experimentally confirmed that f16 + 128K would force CPU spillover.

**Gemma 26B-A4B** has the smallest KV footprint of the three thanks to hybrid attention + MoE. Could easily run f16 KV at 64K with 3+ GB free. This is Unsloth's default config — we just followed it.

**Gemma 31B** is genuinely dense (60 attn layers all active) and the biggest of the three. No room for f16 KV at any useful context. q8_0 KV + 32K is the largest config that fits. This is also Unsloth's recommended "start here" config.

### For the A5500 setup

Same GPU architecture (Ampere), same VRAM (24 GB). Numbers should be within 1-2% of what you see here. The A5500 is about 10% faster than A5000 on compute, so generation speeds will tick up slightly:

- Qwen 3.6 27B: ~27-28 tok/s (vs 25 on A5000)
- Gemma 4 26B-A4B: ~110 tok/s (vs 102)
- Gemma 4 31B: ~31-32 tok/s (vs 29)

Everything else — VRAM footprints, KV cache sizes, required quant choices — should be identical.

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
