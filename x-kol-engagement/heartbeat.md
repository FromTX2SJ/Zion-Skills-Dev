---
name: x-kol-engagement-heartbeat
trigger: interval
interval: 60
quiet_hours:
  timezone: America/Los_Angeles
  start: "00:00"
  end: "07:00"
  interval: 180
description: Poll X for KOL tweets, draft proposals, execute approved actions. Runs 24h skill sync & trend analysis.
state_file: memory/heartbeat-state.json
source_url: https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md
---

> ⚠️ **Fetched remotely every cycle.** Never cache locally. Always get latest from the `source_url` above.

# Heartbeat — X KOL Engagement

**Cycle:** 60 min (180 min during quiet hours 00:00–07:00 PT)

## 🎯 Each Cycle, Do This:

1. **24h tasks** — If due: skill file sync + trend analysis → [24-Hour Cycles](#24-hour-cycles)
2. **Pending approvals** — Execute approved proposals, expire stale ones → [Step 0](#step-0--pre-flight)
3. **Poll tweets** — Fetch new KOL tweets via batched search → [Step 1](#step-1--fetch-tweets)
4. **Triage** — Score by priority/relevance/metrics → [Step 2](#step-2--triage)
5. **Draft proposals** — Using MESSAGE.md voice rules → [Step 3](#step-3--draft-proposals)
6. **Push to human** — Via `message_tool`, save to pending queue (non-blocking) → [Step 4](#step-4--push-to-human)
7. **Execute** — Run approved actions with RT/Quote fallback → [Step 5](#step-5--execute)


## Proactive Messaging

Use `message_tool` to push to human. Don't wait for them to check.

**Always message:** proposals ready, trend posts, skill file diffs, auth errors (401/403), rate limits (429), daily summary.
**Don't message:** zero tweets found, successful action execution.

Keep messages concise and actionable. Example:
```
🐦 4 proposals ready
1. REPLY @VitalikButerin — "AA is the unlock..."
2. LIKE @balaboronkov — agent economics thread
3. QUOTE @aeyakovenko — "This changes..."
4. LIKE @VitalikButerin — same thread
Reply: approve all / approve 1,3 / reject all / skip
```


## Heartbeat State

### `memory/heartbeat-state.json`

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
    "replies": 0, "likes": 0, "quote_tweets": 0,
    "retweets": 0, "follows": 0, "bookmarks": 0, "original_posts": 0
  }
}
```

Statuses: `IDLE` | `POLLING` | `DRAFTING` | `AWAITING_APPROVAL` | `EXECUTING` | `SYNCING` | `TRENDING` | `ERROR`

Reset `actions_today` when `today` ≠ current date.


## 24-Hour Cycles

### Skill File Auto-Sync

**State:** `memory/skill-update-state.json` — tracks `last_check_at`, `next_check_at`, per-file `sha256` hashes, and `update_history` (last 20).

**Procedure:**
1. Skip if `now < next_check_at`
2. Fetch each skill file from its GitHub raw URL (see SKILL.md file table)
3. Compute SHA-256 hash, compare with stored hash
4. **No changes** → update timestamps, set next = now + 24h
5. **Changes detected:**
   - Generate human-readable diff summary
   - Overwrite local file with fetched version
   - Update hash in state
   - Reload agent memory for changed files (rules, voice, API ref, etc.)
   - Notify human via `message_tool`: which files changed + summary
6. Set `next_check_at` = now + 24h

### Trend Analysis

**State:** `memory/trend-state.json` — tracks `last_check_at`, `next_check_at`, `last_trends` (4 regions), `post_history` (last 30).

**Procedure:**
1. Skip if `now < next_check_at`
2. Fetch trends from 4 WOEIDs:
   - Global: `GET /2/trends/by/woeid/1`
   - US: `/woeid/23424977`
   - Singapore: `/woeid/23424948`
   - UAE: `/woeid/23424738`
   - Fallback: `GET /1.1/trends/place.json?id=WOEID`
3. Filter for relevance: AI, agents, crypto, DeFi, web3, identity, ZK, protocols in our ecosystem
4. Save to `last_trends`
5. Draft original post proposal connecting most relevant trend(s) to ZION's thesis
6. Send proposal via `message_tool`, save to pending
7. Set `next_check_at` = now + 24h

**Voice for trend posts:** Follow MESSAGE.md Original Post Guidelines. Don't restate the trend — add your unique angle.


## Poll Cycle Steps

### Step 0 — Pre-Flight

`STATUS → POLLING`

1. Run 24h cycles if due (skill sync, trend analysis)
2. Check `pending-proposals.json` → execute any with `status: "approved"` (jump to Step 5)
3. Load watchlist → if empty, log "Ask human to add KOLs", set `IDLE`, END
4. Load poll state & heartbeat state
5. Reset `actions_today` if date rolled over
6. Check daily caps (RULE.md) → skip capped action types
7. If `consecutive_errors` ≥ 3 → double interval (backoff)
8. **Quiet hours** (00:00–07:00 PT): poll every 180 min, skip drafting, bookmark high-priority tweets

### Step 1 — Fetch Tweets

Build batched search queries from watchlist: `(from:user1 OR from:user2 ...) -is:reply -is:retweet`

- ~25 handles per query (512-char limit). Split if >25 users.
- Use `since_id=global_since_id`. First poll: use `start_time` = 1h ago instead.
- `max_results=100`. Limit to 3 pages max per batch.
- Map `author_id` back to watchlist entries
- Update `x-poll-state.json`: `global_since_id` → `meta.newest_id`, increment `poll_count_today`
- On error: log to `errors[]`, increment `consecutive_errors`, set `ERROR`, END

See SKILL.md [Search Recent Tweets] for full API reference.

### Step 2 — Triage

`STATUS → DRAFTING`

0 tweets → log, set `IDLE`, END.

**Scoring:**

| Signal | Weight |
|--------|--------|
| KOL priority (high=3, med=2, low=1) | 3x |
| Engagement metrics | 2x |
| Topic relevance to ZION themes | 2x |
| Recency | 1x |
| Thread starter (original > continuation) | 1x |

Rank by composite score. Take top N within daily budget.

### Step 3 — Draft Proposals

For each prioritized tweet:
1. Load `reply-style-tracker.json` (MESSAGE.md anti-monotony)
2. Pick personality mode ≠ last reply's mode
3. Pick opener category ≠ last reply's opener
4. Check `zion_mention_cooldown` — if > 0, don't mention ZION
5. Apply all MESSAGE.md voice rules
6. Suggest secondary actions (like, bookmark) where appropriate
7. Respect per-cycle limits (RULE.md)

**Proposal format:** See RULE.md proposal format.

### Step 4 — Push to Human

`STATUS → AWAITING_APPROVAL`

**Non-blocking.** Agent does NOT wait for response.

1. Save proposals to `pending-proposals.json`:
   ```json
   {
     "proposals": [{
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
     }]
   }
   ```
2. Send summary to human via `message_tool`
3. Set status `IDLE` — continue to next cycle

**Lifecycle:** `pending → approved → executed` | `pending → rejected` | `pending → edited → approved → executed` | `pending → (24h) → expired`

Max 20 pending. Expire oldest first if exceeded.

### Step 5 — Execute

`STATUS → EXECUTING`

Execute approved proposals in order: bookmarks → likes → replies → quotes → retweets → follows → original posts.

After each action:
- Update `actions_today` counters
- Update `reply-style-tracker.json`
- Mark proposal `executed`
- On 429 → stop execution, log remaining for next cycle

### RT/Quote Fallback

If retweet or quote tweet fails (403/429/any error):

1. Post fallback tweet: `"{draft_text}\n\n@{handle} https://x.com/{handle}/status/{tweet_id}"`
2. For retweet fallback: `"🔁 @handle https://x.com/handle/status/ID"`
3. If fallback also fails → skip, notify human

Does NOT apply to replies, likes, follows (binary success/fail).


## Error Handling

| Error | Action |
|-------|--------|
| 401 | Stop. Message human: credentials invalid. |
| 403 | For RT/Quote: try fallback. Otherwise skip + message human. |
| 429 | Stop cycle. Back off until `x-rate-limit-reset`. Message human. |
| Network/503 | Retry once, then skip cycle. Increment `consecutive_errors`. |
| Empty watchlist | Message human to add KOLs. |
| Malformed state | Re-initialize with defaults. |

**Backoff:** 1-2 errors → normal 60min. 3-5 → 120min. 6+ → 240min + message human.
