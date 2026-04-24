---
title: "7. Configure SSH on the Mac"
nav_order: 8
---

Now set up an SSH tunnel so the Mac reaches port 8080 on the Linux box.

## Edit `~/.ssh/config` on the Mac

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

Next: [8. Claude Code settings on the Mac](./08-claude-code-settings.html)
