# Zion-Skills-Dev

ZION agent skill definitions — fetched remotely by agents on every heartbeat cycle.

**Repository:** [github.com/FromTX2SJ/Zion-Skills-Dev](https://github.com/FromTX2SJ/Zion-Skills-Dev)


## Skills

| Skill | Description | Entry Point |
|-------|-------------|-------------|
| 🐦 **x-kol-engagement** | Monitor crypto KOLs on X, draft engagement replies, execute human-approved interactions as a ZION cofounder. | [`skill.md`](x-kol-engagement/skill.md) |


---


## x-kol-engagement

Autonomous X (Twitter) engagement skill for ZION cofounders. Polls KOL tweets, drafts contextual replies, and executes human-approved actions.


### Skill Files

| File | Description |
|------|-------------|
| [`skill.md`](x-kol-engagement/skill.md) | Core skill definition — setup, API reference, watchlist management, onboarding task list |
| [`heartbeat.md`](x-kol-engagement/heartbeat.md) | Heartbeat loop — per-cycle task list, polling, triage, proposals, execution |
| [`message.md`](x-kol-engagement/message.md) | Voice & persona guide — 5 personality modes, humor, anti-monotony rules |
| [`rule.md`](x-kol-engagement/rule.md) | Governance — approval workflows, rate limits, safety rails, fallback rules |
| [`skill.json`](x-kol-engagement/skill.json) | Package metadata |


### Features

| Feature | Description |
|---------|-------------|
| **KOL Tweet Polling** | Every 60 min (180 min during quiet hours 00:00–07:00 PT), polls X API for new tweets from watched KOLs |
| **Non-Blocking Approval** | Proposals pushed to human via `message_tool`, saved to pending queue — agent doesn't block waiting |
| **5 Personality Modes** | 🔥 Spicy, 🤓 Deep Tech, 😄 Casual, 🧵 Story, 🤔 Philosophical — rotated per reply |
| **Anti-Monotony Rules** | Tracks last 5 openers, modes, lengths — enforces variety, prevents template-robot behavior |
| **48h Skill File Sync** | Fetches all skill files from GitHub every 48 hours, detects diffs via SHA-256, auto-reloads agent memory |
| **Watchlist Auto-Follow** | Adding a KOL to watchlist auto-follows them (implied intent) |
| **RT/Quote Fallback** | If retweet or quote tweet fails → posts as `{content} @handle {tweet_link}` instead |
| **Quiet Hours** | 00:00–07:00 PT: poll-only mode, no proposals, bookmark high-priority tweets for later |
| **Rate Limits** | Per-cycle and daily caps enforced for all action types (replies, likes, quotes, etc.) |


### Local Storage

```
~/.openclaw/skills/zion-skills-dev/x-kol-engagement/
├── SKILL.md, HEARTBEAT.md, MESSAGE.md, RULE.md    # Fetched from GitHub
├── package.json                                     # Metadata
└── memory/
    └── x-kol-engagement/
        ├── heartbeat-state.json    # Cycle state & daily action counters
        ├── x-watchlist.json        # KOL watchlist
        ├── x-poll-state.json       # Poll tracking (since_id, per-user)
        ├── pending-proposals.json  # Non-blocking approval queue
        ├── skill-update-state.json # 48h file sync state
        └── reply-style-tracker.json # Anti-monotony tracking
```

Authentication: managed by [xurl](https://github.com/xdevplatform/xurl) (stored in `~/.xurl`)


### Quick Start

1. Agent reads [`skill.md`](x-kol-engagement/skill.md) and follows the **🎯 What You Need To Do** task list
2. Verifies xurl is installed and authenticated (`xurl auth status`)
3. Fetches own X profile via `xurl /2/users/me`
4. Asks human for initial KOL watchlist → auto-follows each
5. Registers heartbeat task in `~/.openclaw/workspace/HEARTBEAT.md`
6. Every 60 min: fetches [`heartbeat.md`](x-kol-engagement/heartbeat.md) from GitHub and executes it


---


## How It Works

Agents fetch skill files from GitHub raw URLs on every heartbeat cycle — **never cache logic locally**:

```
https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/{skill-name}/{file}
```

This means skill updates take effect immediately on the next cycle without reinstalling.

### File Roles

| File | Role |
|------|------|
| **skill.md** | Onboarding + reference — agent reads this once to set up, then periodically to stay current |
| **heartbeat.md** | Execution loop — fetched and followed every cycle (the "what to do now" file) |
| **message.md** | Voice guide — personality modes, tone calibration, anti-patterns |
| **rule.md** | Governance — what's allowed, what's not, rate limits, approval workflows |
| **skill.json** | Machine-readable metadata |


## License

MIT License — see [LICENSE](LICENSE).
