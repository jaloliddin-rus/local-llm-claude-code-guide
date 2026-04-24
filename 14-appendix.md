---
title: "14. Appendix — what we actually measured"
nav_order: 15
---

The numbers in this guide aren't estimates. Every one was measured on a real RTX A5000 24 GB + Linux Mint 22.3 + ECC disabled. Should transfer cleanly to RTX A5500 (same Ampere architecture, same VRAM) and similar cards.

## Qwen 3.6 27B UD-Q4_K_XL at 128K ctx, q8_0 KV

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

## Gemma 4 26B-A4B UD-Q4_K_XL at 64K ctx, f16 KV

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

## Gemma 4 31B UD-Q4_K_XL at 32K ctx, q8_0 KV

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

## Why the three configs differ

**Qwen** has larger per-layer KV footprint than Gemma at equivalent context. Had to use q8_0 KV to fit 128K. Experimentally confirmed that f16 + 128K would force CPU spillover.

**Gemma 26B-A4B** has the smallest KV footprint of the three thanks to hybrid attention + MoE. Could easily run f16 KV at 64K with 3+ GB free. This is Unsloth's default config — we just followed it.

**Gemma 31B** is genuinely dense (60 attn layers all active) and the biggest of the three. No room for f16 KV at any useful context. q8_0 KV + 32K is the largest config that fits. This is also Unsloth's recommended "start here" config.

## For the A5500 setup

Same GPU architecture (Ampere), same VRAM (24 GB). Numbers should be within 1-2% of what you see here. The A5500 is about 10% faster than A5000 on compute, so generation speeds will tick up slightly:

- Qwen 3.6 27B: ~27-28 tok/s (vs 25 on A5000)
- Gemma 4 26B-A4B: ~110 tok/s (vs 102)
- Gemma 4 31B: ~31-32 tok/s (vs 29)

Everything else — VRAM footprints, KV cache sizes, required quant choices — should be identical.

---

[Back to home](./)
