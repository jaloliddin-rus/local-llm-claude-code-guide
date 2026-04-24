---
title: "8. Claude Code settings on the Mac"
nav_order: 9
---

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

## What each setting does

| Variable | Effect | Safe for cloud? |
|---|---|---|
| `CLAUDE_CODE_ATTRIBUTION_HEADER=0` | Skips metadata header — **critical for local** (otherwise KV cache invalidates every turn and inference drops 5-10×) | ✅ harmless on cloud |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` | Enterprise-grade telemetry block | ✅ |
| `DISABLE_TELEMETRY=1` | OpenTelemetry metrics off | ✅ |
| `DISABLE_ERROR_REPORTING=1` | Sentry crash reports off | ✅ |
| `DISABLE_BUG_COMMAND=1` | Disables `/bug` slash command | ✅ |
| `DISABLE_AUTOUPDATER=1` | Stop auto-updater; use `claude-update` manually | ✅ |

**`DISABLE_PROMPT_CACHING=1`** is also required for local but goes in the shell function, not here — it costs 90% on cloud. See [step 9](./09-shell-functions-mac.html).

---

Next: [9. Shell functions on the Mac](./09-shell-functions-mac.html)
