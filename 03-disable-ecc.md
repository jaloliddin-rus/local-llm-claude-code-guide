---
title: "3. Disable ECC for extra VRAM"
nav_order: 4
---

Workstation cards ship with ECC on, reserving ~1.5 GB. For LLM inference, ECC is overkill — the risk of a cosmic-ray bit flip during a single turn is vanishingly small, and even if it happened you'd just see one weird token. Disabling ECC gets you the full 24 GB instead of ~22.5 GB.

## Check current state

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

Next: [4. Download the models from Unsloth](./04-download-models.html)
