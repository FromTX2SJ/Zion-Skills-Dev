---
name: x-kol-engagement-heartbeat

trigger: interval

interval: 60

quiet_hours:
  timezone: America/Los_Angeles
  start: "00:00"
  end: "07:00"
  interval: 180

description: Poll X API for new KOL tweets, draft engagement proposals, execute approved actions.

state_file: memory/x-kol-engagement/heartbeat-state.json

source_url: https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md

---


> ⚠️ **This file is fetched remotely every cycle.** Always get the latest version from:
> `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md`
> Do NOT cache or hardcode the heartbeat logic locally.


# X KOL Engagement — Heartbeat


**Cycle:** Every 60 minutes (180 minutes during quiet hours 00:00–07:00 PT)

**Purpose:** Fetch new tweets from watched KOLs, draft engagement proposals, push to human via `message_tool` for approval, execute approved actions.


## 📂 State Files

All state files for this skill are stored under `memory/x-kol-engagement/` (context memory). If a state file doesn't exist on first access, create it with the default schema shown in its section below.

Authentication is managed by xurl (stored in `~/.xurl`). No separate credentials file needed.


## 🎯 What To Do Each Cycle

> **Note:** Pre-flight state checks (timing, date rollover, rate limits, backoff, quiet hours) and post-cycle state updates are handled by the **local heartbeat entry** in `~/.openclaw/workspace/HEARTBEAT.md`. This file only describes the **tasks** to execute.

Every heartbeat cycle, execute these tasks in order:

1. **Execute pending approvals** — Check `pending-proposals.json` for approved/expired proposals → [Step 0](#step-0--pre-flight)
2. **Poll for new tweets** — Fetch latest tweets from watched KOLs → [Step 1](#step-1--fetch-new-tweets)
3. **Triage & prioritize** — Score tweets by relevance, priority, metrics → [Step 2](#step-2--triage--prioritize)
4. **Draft proposals** — Create engagement proposals using MESSAGE.md voice rules → [Step 3](#step-3--draft-engagement-proposals)
5. **Push to human** — Send proposals via `message_tool`, save to pending queue — **non-blocking** → [Step 4](#step-4--push-to-human--save-as-pending-non-blocking)
6. **Execute approved actions** — Run approved proposals, with RT/Quote fallback if needed → [Step 5](#step-5--execute-approved-actions)

> ⬇️ Details for each step are in the sections below.


---


## Proactive Messaging


The agent **MUST** proactively push proposals, alerts, and summaries to the human using `message_tool`. Do NOT wait for the human to check — send the message directly.

### When to Message the Human

| Event | Send? | Priority |
|-------|:-----:|----------|
| Engagement proposals ready | ✅ YES | Normal |
| API auth error (401/403) | ✅ YES | 🔴 Urgent |
| Rate limit hit (429) | ✅ YES | ⚠️ Warning |
| Daily summary | ✅ YES | Low |
| Zero tweets found | ❌ NO | — |
| Successful action execution | ❌ NO | — |

### Message Format

For engagement proposals, use the **standard proposal format** defined in [Step 3](#step-3--draft-engagement-proposals) — same format for drafting and sending. See [Step 4](#step-4--push-to-human--save-as-pending-non-blocking) for the full example of how to send a batch to the human.

For non-proposal messages (alerts, summaries), keep them concise and actionable.


---


## Heartbeat State


### State File: `memory/x-kol-engagement/heartbeat-state.json`

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
    "bookmarks": 0,
    "original_posts": 0
  },
  "poll": {
    "last_poll_at": null,
    "global_since_id": null,
    "per_user_latest": {},
    "poll_count_today": 0,
    "errors": []
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
| `today` | Current date string — reset `actions_today` and `poll.poll_count_today` when date changes |
| `actions_today` | Running daily totals for rate limit enforcement (see RULE.md) |
| `poll.last_poll_at` | Timestamp of last successful tweet poll |
| `poll.global_since_id` | Highest tweet ID seen across all users — used as `since_id` in next search query |
| `poll.per_user_latest` | Per-user tracking: `{ "user_id": { handle, latest_tweet_id, latest_tweet_at } }` |
| `poll.poll_count_today` | Number of polls executed today (reset when `today` changes) |
| `poll.errors` | Last 5 poll errors for debugging |


---


## Poll Cycle Steps


### Step 0 — Pre-Flight

> **Note:** State checks (timing, date rollover, rate limits, backoff, quiet hours) have already been handled by the local heartbeat entry before this file is fetched. This step only handles task-level pre-flight.

1. **Check pending proposals** — load `memory/x-kol-engagement/pending-proposals.json`. If there are any with `status: "approved"`, execute them first (jump to Step 5).
2. **Load watchlist** from `memory/x-kol-engagement/x-watchlist.json`
   - If watchlist is empty → log "Watchlist empty. Ask human to add KOLs." → END


### Step 1 — Fetch New Tweets

Build batched search queries from the watchlist.

**Query construction:**

```
(from:user1 OR from:user2 OR from:user3 ... OR from:userN) -is:reply -is:retweet
```

**Rules:**
- Max ~25 handles per query (due to 512-char query limit on Basic tier)
- If watchlist has > 25 users, split into multiple batched queries
- **Sort watchlist by `priority` first** (high → medium → low), so the first batch always contains the most important KOLs. Within the same priority, sort by handle length to pack queries efficiently.
- ⚠️ **MUST execute ALL batched queries to cover the ENTIRE watchlist.** Do NOT stop after the first batch. Every KOL in the watchlist must be polled every cycle.

**API call:**

```bash
xurl "/2/tweets/search/recent?query=(from:handle1 OR from:handle2 ...) -is:reply -is:retweet&since_id=$GLOBAL_SINCE_ID&max_results=100&tweet.fields=id,text,author_id,created_at,public_metrics,referenced_tweets,entities,conversation_id&expansions=author_id,referenced_tweets.id&user.fields=name,username"
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
- Update the `poll` sub-object in `memory/x-kol-engagement/heartbeat-state.json`:
  - Set `poll.global_since_id` to `meta.newest_id` (highest tweet ID across all batches)
  - Update `poll.per_user_latest` for each author
  - Increment `poll.poll_count_today`
  - Set `poll.last_poll_at` to now

**On error:**
- Append error to `poll.errors[]` in `memory/x-kol-engagement/heartbeat-state.json` (keep last 5)
- Increment `consecutive_errors` (top-level field in same file)
- Set status `ERROR` → END (will retry next cycle)


### Step 2 — Triage & Prioritize

If Step 1 returned 0 tweets:
- Log "No new tweets from watched KOLs"
- END (the local heartbeat entry will handle post-cycle state updates)

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

For each prioritized tweet, draft a structured engagement proposal.

**Before drafting each reply:**
1. Load `memory/x-kol-engagement/reply-style-tracker.json` (see MESSAGE.md Anti-Monotony Rules)
2. Pick a personality mode different from the last reply
3. Pick an opener category different from the last reply
4. Check `zion_mention_cooldown` — if > 0, do NOT mention ZION
5. Apply all MESSAGE.md voice rules

**Proposal format:**

````
────────────────────────────────────────
📋 PROPOSAL #1 — @VitalikButerin
────────────────────────────────────────
🎯 ACTION:      REPLY
🐦 TWEET:       "Just shipped a new feature for account abstraction..."
🔗 TWEET_ID:    1234567890
🔗 LINK:        https://x.com/VitalikButerin/status/1234567890
📊 METRICS:     ❤️ 150  🔁 30  💬 25  📝 10
🏷️ TAGS:        ethereum, founder
⚡ PRIORITY:    high
🎭 MODE:        🤓 Deep Tech

💬 DRAFT REPLY:
```
This is a great step for UX in crypto. Account
abstraction is exactly the kind of infra that makes
autonomous agents viable on-chain. We're building
similar composability into ZION's agent identity layer.
```

📝 REASON:
High-priority KOL discussing account abstraction,
directly relevant to ZION's agent identity architecture.
Authentic technical engagement opportunity.
────────────────────────────────────────
````

**Also suggest secondary actions per tweet where appropriate:**
- 👍 LIKE — always suggest alongside reply/quote
- 🔖 BOOKMARK — for reference-worthy content (auto-approved)

**Respect per-cycle limits from RULE.md:**
- Max 15 reply proposals per cycle
- Max 9 quote tweet proposals per cycle
- Max 30 like proposals per cycle
- Max 6 follow proposals per cycle


### Step 4 — Push to Human & Save as Pending (Non-Blocking)

**This step is NON-BLOCKING.** The agent does NOT wait for human response before continuing to the next poll cycle.

1. **Save proposals** to `memory/x-kol-engagement/pending-proposals.json`:
   ```json
   {
     "proposals": [
       {
         "id": "prop-2025-03-01-001",
         "created_at": "2025-03-01T15:30:00Z",
         "expires_at": "2025-03-02T15:30:00Z",
         "type": "reply",
         "target_handle": "VitalikButerin",
         "target_tweet_id": "1234567890",
         "target_tweet_text": "Just shipped...",
         "draft_text": "This is a great step...",
         "personality_mode": "deep_tech",
         "status": "pending",
         "batch_id": "batch-2025-03-01-1530"
       }
     ]
   }
   ```

2. **Send to human** via `message_tool` — use the **exact same proposal format** from Step 3. Add a header line (`🐦 X KOL Engagement — N proposals ready`) and approval instructions at the end (`Reply: approve all / approve 1,3 / reject all / edit 1: [text] / skip`). See RULE.md §1 for the full list of approval commands.

3. **On next cycle**, check `pending-proposals.json`:
   - If human has responded with approvals → execute approved proposals in Step 5
   - If human has responded with edits → update draft text, mark as approved
   - If proposals are > 24 hours old (`expires_at` passed) → mark as `expired`, remove from pending
   - If human hasn't responded → continue with new poll cycle, new proposals get appended

### Pending Proposals Lifecycle

```
pending → approved → executed
pending → rejected → (removed)
pending → edited → approved → executed
pending → (24h passed) → expired → (removed)
```

**Max pending proposals:** 20. If queue exceeds 20, expire oldest pending proposals first.


### Step 5 — Execute Approved Actions

For each approved proposal, execute in this order:

1. **Bookmarks first** (auto-approved, private, no risk)
2. **Likes** (low visibility, low risk)
3. **Replies** (high visibility, draft was approved)
4. **Quote tweets** (highest visibility)
5. **Retweets** (amplification)
6. **Follows** (relationship building)
7. **Original posts**

**Execution template (Reply example):**

```bash
xurl -X POST /2/tweets -d '{"text":"APPROVED_REPLY_TEXT","reply":{"in_reply_to_tweet_id":"ORIGINAL_TWEET_ID"}}'
```

**After each action:**
- Log success/failure
- Update `actions_today` counters in heartbeat state
- Update `memory/x-kol-engagement/reply-style-tracker.json`:
  - Record the mode, opener, and length used
  - If reply/quote **mentions ZION** → set `zion_mention_cooldown = 2`
  - If reply/quote **does NOT mention ZION** → set `zion_mention_cooldown = max(0, cooldown - 1)`
  - Update `question_deficit`, `humor_deficit`, `minimalist_used_today` per MESSAGE.md Anti-Monotony Rules
- Mark proposal as `executed` in `pending-proposals.json`
- If any action fails with rate limit (429), stop execution and log remaining actions for next cycle

**After all actions complete**, return control to the local heartbeat entry — it handles updating `last_result`, `consecutive_errors`, `status`, `last_run_at`, `next_run_at`, and `cycle_count`.


### Retweet / Quote Tweet Fallback

If a retweet or quote tweet fails, apply the fallback strategy defined in **RULE.md §5** (Retweet / Quote Tweet Fallback). In short: post as a regular tweet with `"{draft_text}\n\n@handle https://x.com/handle/status/ID"` instead.


---


## Error Handling

For API error responses and edge cases, see **RULE.md §5** (Error & Edge Case Handling).

**Backoff strategy** is managed by the local heartbeat entry (see SKILL.md "Set Up Your Heartbeat" §1 Pre-flight).


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
║  📬 Pending:      2                 ║
╠══════════════════════════════════════╣
║  Today's Totals                     ║
║  💬 Replies:    8/60                ║
║  ❤️ Likes:     22/150              ║
║  📝 Quotes:    3/30                ║
║  🔁 Retweets:  1/60                ║
║  👥 Follows:   2/15                ║
║  📢 Posts:     1/9                  ║
╚══════════════════════════════════════╝
```

Send the daily summary to human via `message_tool` at the end of the last cycle before quiet hours begin.
