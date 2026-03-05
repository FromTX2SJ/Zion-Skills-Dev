# Zion-Skills-Dev

ZION agent skill definitions — fetched remotely by agents on every heartbeat cycle.


## Skills

| Skill | Description | Entry Point |
|-------|-------------|-------------|
| **x-kol-engagement** | Monitor crypto KOLs on X, draft engagement replies, execute human-approved interactions as a ZION cofounder. | [`skill.md`](x-kol-engagement/skill.md) |


## How It Works

Each skill is a set of Markdown files that define an agent's behavior:

- **SKILL.md** — Core skill definition, API reference, setup instructions
- **HEARTBEAT.md** — Periodic task loop (fetched remotely every cycle)
- **MESSAGE.md** — Voice & persona guide
- **RULE.md** — Governance rules, rate limits, safety rails
- **skill.json** — Package metadata

Agents fetch these files from GitHub raw URLs on every heartbeat cycle to always run the latest version:

```
https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/{skill-name}/{file}
```


## License

MIT License — see [LICENSE](LICENSE).
