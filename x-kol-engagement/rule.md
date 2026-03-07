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
| **Follow (on watchlist add)** | ✅ AUTO | Human's "add @handle" implies intent to follow |
| **Original post** | 🔴 YES | Public, represents ZION brand |
| **Bookmark** | ✅ AUTO | Private — no public visibility |
| **Search tweets** | ✅ AUTO | Read-only |
| **Lookup users** | ✅ AUTO | Read-only |


### Proposal Format

> **Draft text MUST be wrapped in a code block (triple backticks) so the human can one-tap copy it in Telegram.** This applies to all draft text in proposals — replies, quote tweets, and original posts.
>
> **Every proposal MUST include a direct link to the original tweet** (`https://x.com/{handle}/status/{tweet_id}`) so the human can review the full context before approving.

When presenting actions for approval, use the **standard proposal format** (same as HEARTBEAT.md Step 3). Every proposal uses this structure:

````
────────────────────────────────────────
📋 PROPOSAL #1 — @handle
────────────────────────────────────────
🎯 ACTION:      REPLY | LIKE | QUOTE | RETWEET | FOLLOW
🐦 TWEET:       "First 100 chars of their tweet..."
🔗 TWEET_ID:    1234567890
🔗 LINK:        https://x.com/handle/status/TWEET_ID
📊 METRICS:     ❤️ 150  🔁 30  💬 25  📝 10
🏷️ TAGS:        tag1, tag2
⚡ PRIORITY:    high | medium | low
🎭 MODE:        🤓 Deep Tech (for replies/quotes only)

💬 DRAFT REPLY:
```
Your drafted reply text here.
```

📝 REASON:
1-sentence justification — why this engagement matters.
────────────────────────────────────────
````

For batch proposals, send all with a header and approval instructions at the end:

````
🐦 X KOL Engagement — 3 proposals ready

────────────────────────────────────────
📋 PROPOSAL #1 — @VitalikButerin
────────────────────────────────────────
🎯 ACTION:      REPLY
🐦 TWEET:       "Just shipped a new feature for..."
🔗 TWEET_ID:    1234567890
🔗 LINK:        https://x.com/VitalikButerin/status/1234567890
📊 METRICS:     ❤️ 150  🔁 30  💬 25  📝 10
🏷️ TAGS:        ethereum, founder
⚡ PRIORITY:    high
🎭 MODE:        🤓 Deep Tech

💬 DRAFT REPLY:
```
This is really interesting because...
```

📝 REASON:
High-priority KOL discussing AI agents.
────────────────────────────────────────

────────────────────────────────────────
📋 PROPOSAL #2 — @aaboronkov
────────────────────────────────────────
🎯 ACTION:      LIKE
🐦 TWEET:       "Thread on autonomous agent economics..."
🔗 LINK:        https://x.com/aaboronkov/status/1234567891

📝 REASON:
Relevant to ZION's SHP model, signal boost.
────────────────────────────────────────

────────────────────────────────────────
📋 PROPOSAL #3 — @balaboronkov
────────────────────────────────────────
🎯 ACTION:      QUOTE
🐦 TWEET:       "The future of agent-to-agent..."
🔗 TWEET_ID:    1234567892
🔗 LINK:        https://x.com/balaboronkov/status/1234567892
📊 METRICS:     ❤️ 80  🔁 15  💬 12  📝 5
🏷️ TAGS:        agents, infra
⚡ PRIORITY:    high
🎭 MODE:        🔥 Spicy

💬 DRAFT QUOTE:
```
We've been building exactly this at ZION...
```

📝 REASON:
Direct alignment with ZION mission.
────────────────────────────────────────

Reply: approve all / approve 1,3 / reject 2 / edit 1: [new text] / reject all
````


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
| Original Posts | 1 |
| Bookmarks | No limit |


### Daily Caps

Reset at 00:00 UTC. Tracked in `memory/x-kol-engagement/heartbeat-state.json`.

| Action | Max Per Day |
|--------|:----------:|
| Replies | 20 |
| Likes | 50 |
| Quote Tweets | 10 |
| Retweets | 20 |
| Follows | 5 |
| Original Posts | 3 |
| Bookmarks | No limit |


### Enforcement

- **Before proposing** any action, check remaining daily quota
- **If daily cap is hit**, inform the human: "Daily reply limit reached (20/20). Skipping reply proposals until tomorrow."
- **Never exceed caps**, even if the human explicitly asks — explain the rate limit risk:
  > "X API rate limits are strict. Exceeding them risks account suspension. I'll queue this for tomorrow."
- Track all executed actions in `memory/x-kol-engagement/heartbeat-state.json` under `actions_today`


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


### Retweet / Quote Tweet Fallback

If a **retweet** or **quote tweet** fails (403, 429, or any error), the agent MUST attempt a fallback:

1. **Construct a fallback tweet:** `"{draft_text}\n\n@{handle} https://x.com/{handle}/status/{tweet_id}"`
2. **Post as a regular tweet** via `POST /2/tweets` (no `reply` or `quote_tweet_id` params)
3. **Log:** `⚠️ Quote/RT failed → fallback posted as mention tweet`
4. **If fallback also fails** → skip, log error, notify human

| Original Action | Fallback Format |
|----------------|----------------|
| Retweet | `"🔁 @handle https://x.com/handle/status/ID"` |
| Quote tweet | `"{draft_text}\n\n@handle https://x.com/handle/status/ID"` |

This fallback does NOT apply to replies, likes, or follows (they are binary — succeed or fail).


### Edge Cases

| Scenario | Rule |
|----------|------|
| KOL deletes tweet before you reply | Skip — do not reply to deleted content |
| KOL blocks your account | Remove from watchlist, notify human |
| Tweet is in non-English language | Skip unless you're confident in translation |
| Tweet mentions ZION directly | High priority — always propose engagement |
| Tweet is a thread (conversation_id ≠ tweet_id) | Reply to the thread starter, not individual parts |
| Human hasn't responded to proposals in 3+ cycles | Pause engagement proposals, continue polling silently |
| Pending proposals > 24 hours old | Auto-expire, remove from queue |
| Pending proposals queue > 20 | Expire oldest pending proposals first |


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