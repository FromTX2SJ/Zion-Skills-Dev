# Governance Rules


Rules governing approval workflows, rate limits, safety rails, and watchlist management for X KOL engagement.


---


## 1. Human Approval Workflow


### Golden Rule

> **Every X API write action that is publicly visible MUST be approved by your human before execution. No exceptions.**


### Actions Requiring Approval

| Action | Approval Required | Reason |
|--------|:-----------------:|--------|
| **Reply** | 🔴 YES | Public, represents ZION brand |
| **Quote Tweet** | 🔴 YES | Public, represents ZION brand |
| **Like** | 🔴 YES | Publicly visible on profile |
| **Retweet** | 🔴 YES | Publicly visible on profile |
| **Follow** | 🔴 YES | Publicly visible, signals alignment |
| **Bookmark** | ✅ AUTO | Private — no public visibility |
| **Search tweets** | ✅ AUTO | Read-only |
| **Lookup users** | ✅ AUTO | Read-only |


### Proposal Format

When presenting actions for approval, use this structured format:

```
━━━ ENGAGEMENT PROPOSAL ━━━

🎯 Target: @handle (priority: high)
📝 Tweet: "First 100 chars of their tweet..."
🔗 Link: https://x.com/handle/status/TWEET_ID

Action: REPLY
Draft:
  "Your drafted reply text here."

Reason: [1-sentence justification — why this engagement matters]

━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

For batch proposals (multiple actions from one poll cycle), group and number them:

```
━━━ ENGAGEMENT PROPOSALS (3 actions) ━━━

1/3 — REPLY to @VitalikButerin
📝 "Just shipped a new feature for..."
Draft: "This is really interesting because..."
Reason: High-priority KOL discussing AI agents

2/3 — LIKE @aaboronkov's tweet
📝 "Thread on autonomous agent economics..."
Reason: Relevant to ZION's SHP model, signal boost

3/3 — QUOTE @balaboronkov
📝 "The future of agent-to-agent..."
Draft: "We've been building exactly this at ZION..."
Reason: Direct alignment with ZION mission

━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reply with:
• "approve all" — execute all
• "approve 1,3" — execute selected
• "reject 2" — skip selected
• "edit 1: new text here" — modify and approve
• "reject all" — skip everything
```


### Approval Commands

| Command | Effect |
|---------|--------|
| `approve all` | Execute all proposed actions |
| `approve N` or `approve N,M` | Execute only specified proposals |
| `reject N` or `reject all` | Skip specified or all proposals |
| `edit N: new text` | Replace draft text for proposal N, then execute |
| `hold` | Save proposals, don't execute — revisit later |


### Rules

- **Never auto-execute** a write action, even if the human previously approved a similar one
- **Never combine approval steps** — present all proposals, wait for response, then execute
- **If the human is unresponsive**, do NOT retry or re-prompt. Bookmark interesting tweets for later and move on
- **Expired proposals** — if a tweet is > 24 hours old by the time approval comes, warn the human it may look stale


---


## 2. Rate Limits


### Per-Cycle Limits

Enforced per heartbeat poll cycle (default: 60 min).

| Action | Max Per Cycle |
|--------|:------------:|
| Replies | 5 |
| Likes | 10 |
| Quote Tweets | 3 |
| Retweets | 5 |
| Follows | 2 |
| Bookmarks | No limit |


### Daily Caps

Reset at 00:00 UTC. Tracked in `memory/heartbeat-state.json`.

| Action | Max Per Day |
|--------|:----------:|
| Replies | 20 |
| Likes | 50 |
| Quote Tweets | 10 |
| Retweets | 20 |
| Follows | 5 |
| Bookmarks | No limit |


### Enforcement

- **Before proposing** any action, check remaining daily quota
- **If daily cap is hit**, inform the human: "Daily reply limit reached (20/20). Skipping reply proposals until tomorrow."
- **Never exceed caps**, even if the human explicitly asks — explain the rate limit risk:
  > "X API rate limits are strict. Exceeding them risks account suspension. I'll queue this for tomorrow."
- Track all executed actions in `memory/heartbeat-state.json` under `actions_today`


### X API Platform Rate Limits (Reference)

These are X's own enforced limits — hitting them returns HTTP 429:

| Endpoint | App Limit | User Limit |
|----------|-----------|------------|
| `POST /2/tweets` | 100/15min | 100/15min |
| `POST /2/users/:id/likes` | — | 100/15min |
| `POST /2/users/:id/retweets` | — | 50/15min |
| `POST /2/users/:id/following` | — | 15/15min |
| `GET /2/tweets/search/recent` | 60/15min | 60/15min |

Our daily caps are deliberately **far below** these limits to avoid triggering rate limiters or appearing bot-like.


---


## 3. Safety Rails


### Content Prohibitions

**NEVER engage with or create content involving:**

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
4. ✅ Could this be misinterpreted as financial advice? → If yes, don't post
5. ✅ Does this respect the other person's expertise and position?
6. ✅ Is this tweet < 24 hours old? → If older, reconsider relevance


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


## 5. Error & Edge Case Handling


### API Errors

| Error | Action |
|-------|--------|
| `401 Unauthorized` | Stop all operations. Alert human: credentials may be expired/revoked |
| `403 Forbidden` | Log and skip this action. May indicate blocked by target user |
| `429 Too Many Requests` | Stop poll cycle immediately. Double next poll interval. Alert human |
| `503 Service Unavailable` | Retry once after 60s. If still failing, skip cycle |
| Network timeout | Retry once. If failing, skip cycle, increment `consecutive_errors` |


### Edge Cases

| Scenario | Rule |
|----------|------|
| KOL deletes tweet before you reply | Skip — do not reply to deleted content |
| KOL blocks your account | Remove from watchlist, notify human |
| Tweet is in non-English language | Skip unless you're confident in translation |
| Tweet mentions ZION directly | High priority — always propose engagement |
| Tweet is a thread (conversation_id ≠ tweet_id) | Reply to the thread starter, not individual parts |
| Human hasn't responded to proposals in 3+ cycles | Pause engagement proposals, continue polling silently |


---


## 6. Logging & Transparency


### Action Log

After executing any approved action, log it:

```
✅ Executed: REPLY to @VitalikButerin
   Tweet ID: 1234567890
   Our reply ID: 1234567899
   Time: 2025-03-01T16:45:00Z
```

### Daily Summary

At the end of each day (or when asked), provide a summary:

```
━━━ DAILY SUMMARY (2025-03-01) ━━━

📊 Polls: 12 cycles
📝 Proposals drafted: 8
✅ Actions executed: 5 (3 replies, 1 like, 1 quote)
❌ Rejected by human: 2
⏭️ Skipped (safety): 1

📈 Top engagement:
   Reply to @VitalikButerin — 23 likes, 5 replies

🔑 API calls: ~15 (12 search + 2 user lookup + 1 retry)

━━━━━━━━━━━━━━━━━━━━━━━━━━━
```


---


## Quick Reference: Decision Tree

```
New tweet from watched KOL
  │
  ├─ Is topic in prohibited list? → SKIP
  │
  ├─ Is tweet > 24 hours old? → SKIP (or bookmark)
  │
  ├─ Does it fail self-censorship checks? → SKIP
  │
  ├─ Are we at daily cap for this action type? → SKIP + notify human
  │
  ├─ Draft engagement
  │   ├─ Apply MESSAGE.md voice rules
  │   ├─ Apply anti-patterns check
  │   └─ Format as proposal
  │
  └─ Present to human → WAIT for approval → Execute (or skip)