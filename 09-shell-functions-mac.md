---
title: "9. Shell functions on the Mac"
nav_order: 10
---

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

## The critical design choice

`claude-local` **auto-detects the running model** by querying `/v1/models` and sets `ANTHROPIC_MODEL` accordingly. This means one function works whether you're running Qwen, Gemma 26B, or Gemma 31B — you just start a different model on the Linux side and `claude-local` picks it up on the next run.

`claude-cloud` uses `env -u` to **strip** local-only env vars, so cloud calls never get confused.

---

Next: [10. End-to-end test](./10-end-to-end-test.html)
