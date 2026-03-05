---
name: x-kol-engagement-heartbeat

trigger: interval

interval: 60

description: Poll X API for new KOL tweets, draft engagement proposals, execute approved actions.

state_file: memory/heartbeat-state.json

source_url: https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md

---


> ⚠️ **This file is fetched remotely every cycle.** Always get the latest version from:
> `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md`
> Do NOT cache or hardcode the heartbeat logic locally.


# X KOL Engagement — Heartbeat


**Cycle:** Every 60 minutes (configurable via `interval` above)

**Purpose:** Fetch new tweets from watched KOLs, draft engagement proposals, present to human for approval, execute approved actions.


---


## Heartbeat State


### State File: `memory/heartbeat-state.json`

```json
{
  "status": "IDLE",
  "last_run_at": null,
  "next_run_at": null,
  "last_result": null,
  "consecutive_errors": 0,
  "cycle_count": 0,
  "today": "2025-03-01",
  "actions_today": {
    "replies": 0,
    "likes": 0,
    "quote_tweets": 0,
    "retweets": 0,
    "follows": 0,
    "bookmarks": 0
  }
}
```

| Field | Description |
|-------|-------------|
| `status` | `IDLE`, `POLLING`, `DRAFTING`, `AWAITING_APPROVAL`, `EXECUTING`, `ERROR` |
| `last_run_at` | ISO 8601 timestamp of last completed cycle |
| `next_run_at` | ISO 8601 timestamp of next scheduled cycle |
| `last_result` | Summary of last cycle: `{ tweets_found, proposals_drafted, actions_executed }` |
| `consecutive_errors` | Error counter — back off after 3 consecutive errors |
| `cycle_count` | Total cycles since agent started |
| `today` | Current date string — reset `actions_today` when date changes |
| `actions_today` | Running daily totals for rate limit enforcement (see RULE.md) |


---


## Poll Cycle Steps


### Step 0 — Pre-Flight Checks

```
STATUS → POLLING
```

1. **Load watchlist** from `memory/x-watchlist.json`
   - If watchlist is empty → log "Watchlist empty. Ask human to add KOLs." → set status `IDLE` → END
2. **Load poll state** from `memory/x-poll-state.json`
3. **Load heartbeat state** from `memory/heartbeat-state.json`
4. **Check date rollover** — if `today` ≠ current date, reset `actions_today` counters to 0
5. **Check daily rate limits** — if any counter is at daily cap (see RULE.md), skip those action types this cycle
6. **Check consecutive errors** — if ≥ 3, double the interval (backoff). Log warning.


### Step 1 — Fetch New Tweets

Build batched search queries from the watchlist.

**Query construction:**

```
(from:user1 OR from:user2 OR from:user3 ... OR from:userN) -is:reply -is:retweet
```

**Rules:**
- Max ~25 handles per query (due to 512-char query limit on Basic tier)
- If watchlist has > 25 users, split into multiple queries
- Sort by handle length to pack queries efficiently

**API call:**

```bash
curl -G "https://api.x.com/2/tweets/search/recent" \
  --data-urlencode "query=(from:handle1 OR from:handle2 ...) -is:reply -is:retweet" \
  --data-urlencode "since_id=$GLOBAL_SINCE_ID" \
  --data-urlencode "max_results=100" \
  --data-urlencode "tweet.fields=id,text,author_id,created_at,public_metrics,referenced_tweets,entities,conversation_id" \
  --data-urlencode "expansions=author_id,referenced_tweets.id" \
  --data-urlencode "user.fields=name,username" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
```

**First poll (no `global_since_id`):**
- Add `start_time` param set to 1 hour ago (ISO 8601) instead of `since_id`
- This prevents fetching the entire 7-day search window

**Pagination:**
- If `meta.next_token` exists, fetch next page (same query + `next_token` param)
- Limit to 3 pages max per query batch to control costs

**On success:**
- Collect all tweets into a unified list
- Map each tweet's `author_id` back to the watchlist entry (handle, tags, priority, notes)
- Update `memory/x-poll-state.json`:
  - Set `global_since_id` to `meta.newest_id` (highest tweet ID across all batches)
  - Update `per_user_latest` for each author
  - Increment `poll_count_today`
  - Set `last_poll_at` to now

**On error:**
- Log error to `poll_state.errors[]` (keep last 5)
- Increment `consecutive_errors`
- Set status `ERROR` → END (will retry next cycle)


### Step 2 — Triage & Prioritize

```
STATUS → DRAFTING
```

If Step 1 returned 0 tweets:
- Log "No new tweets from watched KOLs"
- Set `last_result = { tweets_found: 0, proposals_drafted: 0, actions_executed: 0 }`
- Set status `IDLE` → END

If tweets were found, score and rank them:

**Scoring criteria:**

| Signal | Weight | Description |
|--------|--------|-------------|
| KOL priority | 3x | `high` = 3, `medium` = 2, `low` = 1 |
| Engagement metrics | 2x | Higher like/RT counts = more visibility for our reply |
| Topic relevance | 2x | Matches ZION themes: AI agents, crypto, autonomous networks, DeFi |
| Recency | 1x | Newer tweets preferred (more likely to be seen) |
| Thread starter | 1x | Original tweets > thread continuations |

**Rank all tweets by composite score.** Take top N based on remaining daily budget.


### Step 3 — Draft Engagement Proposals

For each prioritized tweet, draft a structured engagement proposal:

```
────────────────────────────────────────
📋 PROPOSAL #1 — @VitalikButerin
────────────────────────────────────────
🎯 ACTION:      REPLY
🐦 TWEET:       "Just shipped a new feature for account abstraction..."
🔗 TWEET_ID:    1234567890
📊 METRICS:     ❤️ 150  🔁 30  💬 25  📝 10
🏷️ TAGS:        ethereum, founder
⚡ PRIORITY:    high

💬 DRAFT REPLY:
"This is a great step for UX in crypto. Account
abstraction is exactly the kind of infra that makes
autonomous agents viable on-chain. We're building
similar composability into ZION's agent identity layer."

📝 REASON:
High-priority KOL discussing account abstraction,
directly relevant to ZION's agent identity architecture.
Authentic technical engagement opportunity.
────────────────────────────────────────
```

**Also suggest secondary actions per tweet where appropriate:**
- 👍 LIKE — always suggest alongside reply/quote
- 🔖 BOOKMARK — for reference-worthy content (auto-approved)

**Respect per-cycle limits from RULE.md:**
- Max 5 reply proposals per cycle
- Max 3 quote tweet proposals per cycle
- Max 10 like proposals per cycle
- Max 2 follow proposals per cycle


### Step 4 — Present to Human

```
STATUS → AWAITING_APPROVAL
```

Present all proposals in a numbered list. Ask the human to:

```
Review the proposals above and respond with:
  ✅ "approve all"         — execute all proposals
  ✅ "approve 1,3,5"       — execute specific proposals by number
  ❌ "reject all"          — skip all proposals
  ✏️ "edit 2: [new text]"  — modify draft text for proposal #2
  ⏭️ "skip"               — skip this cycle entirely
```

**Wait for human response.** Do NOT proceed without approval. Do NOT time out and auto-execute.

**If human edits a proposal:** update the draft text and re-display for final confirmation.


### Step 5 — Execute Approved Actions

```
STATUS → EXECUTING
```

For each approved proposal, execute in this order:

1. **Bookmarks first** (auto-approved, private, no risk)
2. **Likes** (low visibility, low risk)
3. **Replies** (high visibility, draft was approved)
4. **Quote tweets** (highest visibility)
5. **Retweets** (amplification)
6. **Follows** (relationship building)

**Execution template (Reply example):**

```bash
curl -X POST "https://api.x.com/2/tweets" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{
    "text": "APPROVED_REPLY_TEXT",
    "reply": {
      "in_reply_to_tweet_id": "ORIGINAL_TWEET_ID"
    }
  }'
```

**After each action:**
- Log success/failure
- Update `actions_today` counters in heartbeat state
- If any action fails with rate limit (429), stop execution and log remaining actions for next cycle

**After all actions:**
- Set `last_result = { tweets_found: N, proposals_drafted: M, actions_executed: K }`
- Reset `consecutive_errors` to 0
- Set status `IDLE`
- Compute `next_run_at` = now + interval minutes


---


## Error Handling


| Error | Response |
|-------|----------|
| **401 Unauthorized** | Bearer token expired or invalid. Alert human: "X API auth failed. Check credentials." STOP. |
| **403 Forbidden** | App permissions insufficient. Alert human. STOP. |
| **429 Rate Limited** | Log the `x-rate-limit-reset` header. Back off until reset time. Do NOT retry immediately. |
| **Network Error** | Increment `consecutive_errors`. Retry next cycle. |
| **Empty Watchlist** | Prompt human: "No KOLs in watchlist. Use 'add @handle' to start monitoring." |
| **Malformed State** | Re-initialize state file with defaults. Log warning. |


### Backoff Strategy

| Consecutive Errors | Behavior |
|-------------------|----------|
| 1–2 | Normal interval (60 min) |
| 3–5 | Double interval (120 min) |
| 6+ | Quadruple interval (240 min) + alert human |


---


## Cycle Summary Template

After each cycle completes, log a summary:

```
╔══════════════════════════════════════╗
║  X KOL Engagement — Cycle #42       ║
╠══════════════════════════════════════╣
║  🕐 Time:        2025-03-01 15:30   ║
║  📡 Tweets Found: 7                 ║
║  📋 Proposals:    4                 ║
║  ✅ Approved:     3                 ║
║  ❌ Rejected:     1                 ║
║  🚀 Executed:     3                 ║
║  💰 API Calls:    2                 ║
║  ⏭️ Next Poll:    16:30            ║
╠══════════════════════════════════════╣
║  Today's Totals                     ║
║  💬 Replies:    8/20                ║
║  ❤️ Likes:     22/50               ║
║  📝 Quotes:    3/10                ║
║  🔁 Retweets:  1/∞                 ║
║  👥 Follows:   2/5                  ║
╚══════════════════════════════════════╝