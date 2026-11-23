# ai-skills

Agent skills (`SKILL.md`) for AI coding agents such as Claude Code. Each folder is one skill.

| Skill | What it does |
|---|---|
| [network-analyser](network-analyser/SKILL.md) | Fast L2/L3 troubleshooting for MikroTik RouterOS: VLAN, ARP, MAC table, STP path. Read-only analysis, max 2 query rounds, fixes come with backup and rollback. |

## Install (Claude Code)

Copy a skill folder into your skills directory.

Linux/macOS:

```bash
git clone https://github.com/Weed1/ai-skills.git
cp -r ai-skills/network-analyser ~/.claude/skills/
```

Windows PowerShell:

```powershell
git clone https://github.com/Weed1/ai-skills.git
Copy-Item -Recurse ai-skills\network-analyser "$env:USERPROFILE\.claude\skills\"
```

Read a skill before installing it: skills run with your agent's full trust.

## License

MIT
