---
title: "2. Build llama.cpp on the Linux box"
nav_order: 3
---

We use native llama.cpp (not LM Studio's embedded version) because:

- It speaks the Anthropic Messages API natively at `/v1/messages` (since b4847, Jan 2026) — so Claude Code connects with no proxy
- It supports `--chat-template-kwargs` for toggling thinking mode on Qwen and Gemma
- Gets updates the day llama.cpp publishes them

## Install build tools

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake git libcurl4-openssl-dev
```

## Clone and build with CUDA

```bash
cd ~
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

cmake -B build -DGGML_CUDA=ON -DBUILD_SHARED_LIBS=OFF
cmake --build build --config Release -j --target llama-server llama-cli llama-gguf-split llama-mtmd-cli
```

Takes 5-15 minutes. The `llama-mtmd-cli` target is for multimodal (vision) support, which Gemma 4 needs.

## Install binaries on PATH

```bash
sudo install build/bin/llama-* /usr/local/bin/
```

## Verify

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

Next: [3. Disable ECC for extra VRAM](./03-disable-ecc.html)
