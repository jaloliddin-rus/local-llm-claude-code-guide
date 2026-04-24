---
title: "4. Download the models from Unsloth"
nav_order: 5
---

**Don't use LM Studio's `lms get` for this.** Its picker defaults to mirrors like `lmstudio-community` which don't have Unsloth's Dynamic 2.0 quants. Go straight to HuggingFace.

## Install `hf` CLI

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

## Pick a location

Models should live on your NVMe. If you're using LM Studio too, put them under its models directory so LM Studio can see them:

```bash
MODELS_BASE="$HOME/.lmstudio/models"   # or wherever
mkdir -p "$MODELS_BASE"
```

If you're not using LM Studio, any path works. Just remember where you put them for the shell functions in [step 5](./05-shell-functions-linux.html).

## Download the three models

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

Next: [5. Write the llm-* shell functions on Linux](./05-shell-functions-linux.html)
