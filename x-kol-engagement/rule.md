# Governance Rules


Rules governing proposal limits, safety rails, and watchlist management for X KOL engagement.

> **The agent does NOT execute any X API write actions.** It only polls, drafts proposals, and pushes them to the human. These rules govern what the agent should draft and what it should skip.


---


## 1. Proposal Limits


### Per-Cycle Limits

Enforced per heartbeat poll cycle (default: 60 min).

| Metric | Max Per Cycle |
|--------|:------------:|
| Proposals | 15 |


### Daily Cap

Reset at 00:00 UTC. Tracked in `memory/x-kol-engagement/heartbeat-state.json`.

| Metric | Max Per Day |
|--------|:----------:|
| Proposals | 60 |

### Enforcement

- **Before drafting**, check remaining daily quota
- **If daily cap is hit**, log: "Daily proposal limit reached. Skipping drafting until tomorrow."
- Track proposals in `memory/x-kol-engagement/heartbeat-state.json` under `proposals_today`


### X API Platform Rate Limits (Reference)

These are X's own enforced limits for the read endpoints we use:

| Endpoint | App Limit | User Limit |
|----------|-----------|------------|
| `GET /2/tweets/search/recent` | 60/15min | 60/15min |
| `GET /2/users/by` | 300/15min | 300/15min |
| `POST /2/users/:id/following` | — | 15/15min |

The follow endpoint is only used for auto-follow on watchlist add.


---


## 2. Safety Rails


### Content Prohibitions

**NEVER draft proposals for content involving:**

| Category | Examples | Action |
|----------|----------|--------|
| **Political content** | Elections, political figures, partisan takes | Skip entirely |
| **Financial advice** | "Buy X token", price predictions, investment recommendations | Skip entirely |
| **Personal attacks** | Insults, callouts, drama threads | Skip entirely |
| **Controversial threads** | Heated arguments, ratio'd tweets, culture war topics | Skip entirely |
| **Sensitive topics** | Legal disputes, security breaches (unless public & relevant), personal crises | Skip entirely |
| **NSFW content** | Anything adult or explicit | Skip entirely |
| **Scam/spam threads** | Giveaway scams, phishing, pump & dump | Skip entirely |


### Topic Guardrails

| Topic | Allowed? | Notes |
|-------|:--------:|-------|
| AI/ML technical discussion | ✅ | Core territory |
| Autonomous agents & infrastructure | ✅ | Core territory |
| Crypto/DeFi technical architecture | ✅ | Core territory |
| Token prices & market speculation | ❌ | Never |
| Regulatory opinions | ❌ | Never |
| Competitor bashing | ❌ | Never — acknowledge good work |
| ZION technical details | ✅ | Only public info from skill.md |
| ZION token/economics speculation | ❌ | Never speculate |
| Other projects' technical merits | ✅ | Respectful, factual only |


### Self-Censorship Checks

Before finalizing any draft, verify:

1. ✅ Does this add genuine value to the conversation?
2. ✅ Would a real cofounder post this at a conference?
3. ✅ Is this factually accurate based on public information?
4. ✅ Could this be misinterpreted as financial advice? → If yes, don't draft
5. ✅ Does this respect the other person's expertise and position?
6. ✅ Is this tweet < 24 hours old? → If older, reconsider relevance


---


## 3. Proposal Format

> **Draft text MUST be wrapped in a code block (triple backticks) so the human can one-tap copy it.** Every proposal MUST include a direct link to the original tweet (`https://x.com/{handle}/status/{tweet_id}`).

Use the **standard proposal format** defined in **HEARTBEAT.md Step 3**. Key rules:

- Each proposal has: `📋 PROPOSAL #N — @handle` header, ACTION, TWEET, LINK, METRICS, TAGS, PRIORITY, MODE, DRAFT (in code block), REASON
- For batches: add header `🐦 X KOL Engagement — N proposals ready`

See HEARTBEAT.md Step 3 for the complete template and Step 4 for the push flow.


---


## 4. Watchlist Governance


### Who Controls the Watchlist

- **Only the human** can add or remove users from the watchlist
- The agent may **suggest** additions but MUST NOT auto-add


### Suggesting Additions

When you discover a relevant account (e.g., someone engaging with ZION content, or a KOL mentioned by watched users), suggest it:

```
💡 Watchlist Suggestion:
@new_handle — "AI infrastructure builder, 45K followers"
Reason: Frequently engages with @watched_kol on agent topics
Suggested tags: ai-infra, builder
Suggested priority: medium

Add? Reply "add @new_handle" to confirm.
```


### Watchlist Size Limits

| Tier | Max Users | Reasoning |
|------|:---------:|-----------|
| Recommended | ≤ 25 | Fits in single search query |
| Acceptable | 26–50 | Requires 2 batched queries per poll |
| Maximum | 75 | 3 batched queries — cost adds up |
| Hard limit | 100 | Beyond this, polling becomes too expensive |

- If watchlist exceeds 50 users, warn the human about cost implications
- If watchlist exceeds 75 users, strongly recommend pruning low-priority entries


### Pruning Suggestions

Periodically (weekly), suggest pruning inactive accounts:

```
🧹 Watchlist Maintenance:
These accounts haven't posted in 14+ days:
- @inactive_handle1 (last tweet: Feb 15)
- @inactive_handle2 (last tweet: Feb 10)

Remove them to save API costs? Reply "remove @handle" for each.
```


---


## 5. Error Handling


### API Errors

| Error | Action |
|-------|--------|
| `401 Unauthorized` | Stop all operations. Alert human: credentials may be expired/revoked |
| `403 Forbidden` | Log and skip. May indicate blocked by target user |
| `429 Too Many Requests` | Stop poll cycle immediately. Double next poll interval. Alert human |
| `503 Service Unavailable` | Retry once after 60s. If still failing, skip cycle |
| Network timeout | Retry once. If failing, skip cycle, increment `consecutive_errors` |


### Edge Cases

| Scenario | Rule |
|----------|------|
| KOL deletes tweet before you draft | Skip — do not draft for deleted content |
| KOL blocks your account | Remove from watchlist, notify human |
| Tweet is in non-English language | Skip unless confident in translation |
| Tweet mentions ZION directly | High priority — always propose engagement |
| Tweet is a thread (conversation_id ≠ tweet_id) | Draft for the thread starter, not individual parts |


---


## 6. Logging & Transparency


### Daily Summary

At the end of each day (or when asked), provide a summary:

```
━━━ DAILY SUMMARY (2025-03-01) ━━━

📊 Polls: 12 cycles
📋 Proposals drafted: 18
⏭️ Skipped (safety): 3

🔑 API calls: ~13 (12 search + 1 user lookup)

━━━━━━━━━━━━━━━━━━━━━━━━━━━
```


---


## Quick Reference: Decision Tree

```
New tweet from watched KOL
  │
  ├─ Is topic in prohibited list? → SKIP
  │
  ├─ Is tweet > 24 hours old? → SKIP
  │
  ├─ Does it fail self-censorship checks? → SKIP
  │
  ├─ Are we at daily proposal cap? → SKIP + log
  │
  ├─ Draft engagement proposal
  │   ├─ Apply MESSAGE.md voice rules
  │   ├─ Apply anti-patterns check
  │   └─ Format as proposal
  │
  └─ Push to human via message_tool
```
