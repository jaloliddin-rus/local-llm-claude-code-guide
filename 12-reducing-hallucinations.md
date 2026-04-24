---
title: "12. Reducing hallucinations on niche tools"
nav_order: 13
---

Open 27B/31B models have fuzzy recall on niche APIs — specialized command flags, scientific tool options, library internals. Both Qwen and Gemma will confidently invent function names on specialty domains.

**The mitigation that actually works:** give the model a reference file in your project and tell it to `grep` before writing commands. Claude Code will follow this instruction because it has a Bash tool.

## Pattern: project-level CLAUDE.md

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

Next: [13. Troubleshooting](./13-troubleshooting.html)
