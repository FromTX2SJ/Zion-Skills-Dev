---
name: x-kol-engagement-heartbeat

trigger: interval

interval: 60

quiet_hours:
  timezone: America/Los_Angeles
  start: "00:00"
  end: "07:00"
  interval: 180

description: Poll X API for new KOL tweets, draft engagement proposals, push to human.

state_file: memory/x-kol-engagement/heartbeat-state.json

source_url: https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md

---


> ⚠️ **This file is fetched remotely every cycle.** Always get the latest version from:
> `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md`
> Do NOT cache or hardcode the heartbeat logic locally.


# X KOL Engagement — Heartbeat


**Cycle:** Every 60 minutes (180 minutes during quiet hours 00:00–07:00 PT)

**Purpose:** Fetch new tweets from watched KOLs, draft engagement proposals, push to human via `message_tool`.

> **The agent does NOT execute any X API write actions** (no replies, likes, retweets, quotes, bookmarks, or original posts). The agent only polls, drafts, and pushes proposals to the human. The human decides what to do with them.


## 📂 State Files

All state files for this skill are stored under `memory/x-kol-engagement/` (context memory). If a state file doesn't exist on first access, create it with the default schema shown in its section below.

Authentication is managed by xurl (stored in `~/.xurl`). No separate credentials file needed.


## 🎯 What To Do Each Cycle

> **Note:** Pre-flight state checks (timing, date rollover, backoff, quiet hours) and post-cycle state updates are handled by the **local heartbeat entry** in `~/.openclaw/workspace/HEARTBEAT.md`. This file only describes the **tasks** to run each cycle.

Every heartbeat cycle, run these tasks in order:

1. **Poll for new tweets** — Fetch latest tweets from watched KOLs → [Step 1](#step-1--fetch-new-tweets)
2. **Triage & prioritize** — Score tweets by relevance, priority, metrics → [Step 2](#step-2--triage--prioritize)
3. **Draft proposals** — Create engagement proposals using MESSAGE.md voice rules → [Step 3](#step-3--draft-engagement-proposals)
4. **Push to human** — Send proposals via `message_tool` → [Step 4](#step-4--push-to-human)

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

### Message Format

For engagement proposals, use the **standard proposal format** defined in [Step 3](#step-3--draft-engagement-proposals). See [Step 4](#step-4--push-to-human) for how to send a batch to the human.

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
  "proposals_today": 0,
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
| `status` | `IDLE`, `POLLING`, `DRAFTING`, `ERROR` |
| `last_run_at` | ISO 8601 timestamp of last completed cycle |
| `next_run_at` | ISO 8601 timestamp of next scheduled cycle |
| `last_result` | Summary of last cycle: `{ tweets_found, proposals_drafted }` |
| `consecutive_errors` | Error counter — back off after 3 consecutive errors |
| `cycle_count` | Total cycles since agent started |
| `today` | Current date string — reset `proposals_today` and `poll.poll_count_today` when date changes |
| `proposals_today` | Running daily total of proposals pushed to human |
| `poll.last_poll_at` | Timestamp of last successful tweet poll |
| `poll.global_since_id` | Highest tweet ID seen across all users — used as `since_id` in next search query |
| `poll.per_user_latest` | Per-user tracking: `{ "user_id": { handle, latest_tweet_id, latest_tweet_at } }` |
| `poll.poll_count_today` | Number of polls run today (reset when `today` changes) |
| `poll.errors` | Last 5 poll errors for debugging |


---


## Poll Cycle Steps


### Step 1 — Fetch New Tweets

Load watchlist from `memory/x-kol-engagement/x-watchlist.json`.
- If watchlist is empty → log "Watchlist empty. Ask human to add KOLs." → END

Build batched search queries from the watchlist.

**Query construction:**

```
(from:user1 OR from:user2 OR from:user3 ... OR from:userN) -is:reply -is:retweet
```

**Rules:**
- Max ~25 handles per query (due to 512-char query limit on Basic tier)
- If watchlist has > 25 users, split into multiple batched queries
- **Sort watchlist by `priority` first** (high → medium → low), so the first batch always contains the most important KOLs. Within the same priority, sort by handle length to pack queries efficiently.
- ⚠️ **MUST run ALL batched queries to cover the ENTIRE watchlist.** Do NOT stop after the first batch. Every KOL in the watchlist must be polled every cycle.

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
| Engagement metrics | 2x | Higher like/RT counts = more visibility |
| Topic relevance | 2x | Matches ZION themes: AI agents, crypto, autonomous networks, DeFi |
| Recency | 1x | Newer tweets preferred (more likely to be seen) |
| Thread starter | 1x | Original tweets > thread continuations |

**Rank all tweets by composite score.** Take top N (max 15 proposals per cycle).


### Step 3 — Draft Engagement Proposals

For each prioritized tweet, draft a structured engagement proposal.

**Before drafting each reply:**
1. Load `memory/x-kol-engagement/reply-style-tracker.json` (see MESSAGE.md Anti-Monotony Rules)
2. Pick a personality mode different from the last draft
3. Pick an opener category different from the last draft
4. Check `zion_mention_cooldown` — if > 0, do NOT mention ZION
5. Apply all MESSAGE.md voice rules

**Proposal format — use the EXACT template defined in RULE.md Section 3.** You MUST use every field in the template. Do NOT simplify, abbreviate, or omit fields. See [RULE.md § Proposal Format](https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/rule.md) for the canonical template.

**Per-cycle limits:** Max 15 proposals per cycle.

**After drafting**, update `memory/x-kol-engagement/reply-style-tracker.json`:
- Push mode to `last_5_modes`, opener to `last_5_openers`, length to `last_5_lengths` — if array exceeds 5, drop the oldest entry
- **zion_mention_cooldown:** if draft mentions ZION → set to `2`; otherwise → `max(0, cooldown - 1)`
- **question_deficit:** if draft is question-style → reset to `0`; otherwise → `+1`. If reaches `5`, next draft MUST be a question
- **humor_deficit:** if draft contains humor → reset to `0`; otherwise → `+1`. If reaches `8`, next draft MUST include humor
- **minimalist_used_today:** if draft is minimalist style (e.g. "This.", "Underrated.") → set to `true` (max 1 per day)


### Step 4 — Push to Human

Send all proposals to human via `message_tool`. Push the drafts and move on. The human will read them and manually decide what to do on X. **Your job ends after pushing — do NOT ask the human to approve, confirm, or select proposals.**

**Batch format:**

```
🐦 X KOL Engagement — N proposals ready

[Proposal #1]
[Proposal #2]
...

Draft text is in code blocks — tap to copy.
Links included for each tweet.
```

**After pushing:**
- Increment `proposals_today` in heartbeat state
- Return control to the local heartbeat entry for post-cycle state updates


---


## Error Handling

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
║  💰 API Calls:    2                 ║
║  ⏭️ Next Poll:    16:30            ║
╠══════════════════════════════════════╣
║  Today's Totals                     ║
║  📋 Proposals:  12                  ║
║  📡 Polls:      5                   ║
╚══════════════════════════════════════╝
```

Send the daily summary to human via `message_tool` at the end of the last cycle before quiet hours begin.
