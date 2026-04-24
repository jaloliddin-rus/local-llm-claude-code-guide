---
title: "1. Hardware prerequisites"
nav_order: 2
---

## On the Linux box (where models run)

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

## On the Mac (where Claude Code runs)

```bash
claude --version
```

If missing:
```bash
npm install -g @anthropic-ai/claude-code
# if npm is missing: brew install node
```

---

Next: [2. Build llama.cpp on the Linux box](./02-build-llamacpp.html)
