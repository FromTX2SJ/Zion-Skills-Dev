---
name: x-kol-engagement-heartbeat

trigger: interval

interval: 60

quiet_hours:
  timezone: America/Los_Angeles
  start: "00:00"
  end: "07:00"
  interval: 180

description: Poll X API for new KOL tweets, draft engagement proposals, execute approved actions. Also runs 24h cycles for skill file sync and trend analysis.

state_file: memory/heartbeat-state.json

source_url: https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md

---


> ⚠️ **This file is fetched remotely every cycle.** Always get the latest version from:
> `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md`
> Do NOT cache or hardcode the heartbeat logic locally.


# X KOL Engagement — Heartbeat


**Cycle:** Every 60 minutes (180 minutes during quiet hours 00:00–07:00 PT)

**Purpose:** Fetch new tweets from watched KOLs, draft engagement proposals, push to human via `message_tool` for approval, execute approved actions. Also runs daily cycles for skill file updates and trend analysis.


---


## Proactive Messaging


The agent **MUST** proactively push proposals, alerts, and summaries to the human using `message_tool`. Do NOT wait for the human to check — send the message directly.

### When to Message the Human

| Event | Send? | Priority |
|-------|:-----:|----------|
| Engagement proposals ready | ✅ YES | Normal |
| Trend-based post proposal | ✅ YES | Normal |
| Skill files updated (diff detected) | ✅ YES | Low |
| API auth error (401/403) | ✅ YES | 🔴 Urgent |
| Rate limit hit (429) | ✅ YES | ⚠️ Warning |
| Daily summary | ✅ YES | Low |
| Zero tweets found | ❌ NO | — |
| Successful action execution | ❌ NO | — |

### Message Format

Keep messages concise but actionable:

```
🐦 X KOL Engagement — 4 proposals ready

1. REPLY @VitalikButerin — "AA is the unlock..."
2. LIKE @balaboronkov — agent economics thread
3. QUOTE @aeyakovenko — "This changes..."
4. LIKE @VitalikButerin — same thread

Reply: approve all / approve 1,3 / reject all / skip
```


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
    "bookmarks": 0,
    "original_posts": 0
  }
}
```

| Field | Description |
|-------|-------------|
| `status` | `IDLE`, `POLLING`, `DRAFTING`, `AWAITING_APPROVAL`, `EXECUTING`, `SYNCING`, `TRENDING`, `ERROR` |
| `last_run_at` | ISO 8601 timestamp of last completed cycle |
| `next_run_at` | ISO 8601 timestamp of next scheduled cycle |
| `last_result` | Summary of last cycle: `{ tweets_found, proposals_drafted, actions_executed }` |
| `consecutive_errors` | Error counter — back off after 3 consecutive errors |
| `cycle_count` | Total cycles since agent started |
| `today` | Current date string — reset `actions_today` when date changes |
| `actions_today` | Running daily totals for rate limit enforcement (see RULE.md) |


---


## 24-Hour Cycles


Two maintenance tasks run every 24 hours, checked at the start of each heartbeat cycle.


### Skill File Auto-Sync (Every 24 Hours)

```
STATUS → SYNCING
```

**Purpose:** Keep local skill files up-to-date by fetching from GitHub and detecting changes.

**State file:** `memory/skill-update-state.json`

```json
{
  "last_check_at": "2025-03-01T12:00:00Z",
  "next_check_at": "2025-03-02T12:00:00Z",
  "files": {
    "skill.md": {
      "url": "https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.md",
      "local_path": "~/.openclaw/skills/zion-skills-dev/x-kol-engagement/SKILL.md",
      "last_updated_at": "2025-03-01T12:00:00Z",
      "sha256": "abc123...",
      "changed": false
    },
    "heartbeat.md": {
      "url": "https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md",
      "local_path": "~/.openclaw/skills/zion-skills-dev/x-kol-engagement/HEARTBEAT.md",
      "last_updated_at": "2025-03-01T12:00:00Z",
      "sha256": "def456...",
      "changed": false
    },
    "rule.md": {
      "url": "https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/rule.md",
      "local_path": "~/.openclaw/skills/zion-skills-dev/x-kol-engagement/RULE.md",
      "last_updated_at": "2025-03-01T12:00:00Z",
      "sha256": "ghi789...",
      "changed": false
    },
    "message.md": {
      "url": "https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/message.md",
      "local_path": "~/.openclaw/skills/zion-skills-dev/x-kol-engagement/MESSAGE.md",
      "last_updated_at": "2025-03-01T12:00:00Z",
      "sha256": "jkl012...",
      "changed": false
    },
    "skill.json": {
      "url": "https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.json",
      "local_path": "~/.openclaw/skills/zion-skills-dev/x-kol-engagement/package.json",
      "last_updated_at": "2025-03-01T12:00:00Z",
      "sha256": "mno345...",
      "changed": false
    }
  },
  "update_history": []
}
```

**Sync procedure:**

1. **Check timing** — if `now < next_check_at`, skip sync this cycle.
2. **Fetch each file** from its GitHub raw URL.
3. **Compute SHA-256 hash** of fetched content.
4. **Compare** with stored `sha256` for each file.
5. **If no changes** — update `last_check_at`, set `next_check_at` = now + 24h. Done.
6. **If changes detected:**
   a. For each changed file, generate a human-readable diff summary (what sections changed, what's new/removed).
   b. Overwrite the local file with the fetched version.
   c. Update `sha256`, `last_updated_at`, set `changed: true` in state.
   d. **Reload agent memory:**
      - `RULE.md` changed → reload governance rules (rate limits, safety rails, approval workflow)
      - `MESSAGE.md` changed → reload voice guide (personality modes, anti-monotony rules)
      - `HEARTBEAT.md` changed → next cycle will use the new heartbeat logic
      - `SKILL.md` changed → reload API reference, watchlist format, auth config
      - `skill.json` changed → reload metadata
   e. Append to `update_history`:
      ```json
      {
        "at": "2025-03-02T12:00:00Z",
        "files_changed": ["rule.md", "message.md"],
        "summary": "rule.md: Updated daily caps for likes (50→40). message.md: Added new personality mode."
      }
      ```
   f. **Notify human** via `message_tool`:
      ```
      🔄 Skill Files Updated

      Changed files:
      • rule.md — Updated daily caps for likes (50→40)
      • message.md — Added new personality mode

      Changes applied automatically. Review at:
      https://github.com/FromTX2SJ/Zion-Skills-Dev/commits/main
      ```
7. Set `next_check_at` = now + 24h.

**Keep `update_history` to last 20 entries** to avoid unbounded growth.


### Trend Analysis (Every 24 Hours)

```
STATUS → TRENDING
```

**Purpose:** Analyze trending topics globally and in key regions, propose an original post connecting trends to ZION's mission.

**State file:** `memory/trend-state.json`

```json
{
  "last_check_at": "2025-03-01T12:00:00Z",
  "next_check_at": "2025-03-02T12:00:00Z",
  "last_trends": {
    "global": [],
    "us": [],
    "singapore": [],
    "uae": []
  },
  "post_history": []
}
```

**Trend analysis procedure:**

1. **Check timing** — if `now < next_check_at`, skip this cycle.
2. **Fetch trends** from X API:
   ```bash
   # Global trends
   curl "https://api.x.com/2/trends/by/woeid/1" \
     -H "Authorization: Bearer $X_BEARER_TOKEN"

   # United States
   curl "https://api.x.com/2/trends/by/woeid/23424977" \
     -H "Authorization: Bearer $X_BEARER_TOKEN"

   # Singapore (crypto/AI hub)
   curl "https://api.x.com/2/trends/by/woeid/23424948" \
     -H "Authorization: Bearer $X_BEARER_TOKEN"

   # UAE / Dubai (crypto hub)
   curl "https://api.x.com/2/trends/by/woeid/23424738" \
     -H "Authorization: Bearer $X_BEARER_TOKEN"
   ```
   
   **Fallback if v2 Trends unavailable:** Use v1.1 endpoint:
   ```bash
   curl "https://api.x.com/1.1/trends/place.json?id=1" \
     -H "Authorization: Bearer $X_BEARER_TOKEN"
   ```

3. **Filter for relevance** — scan trends for keywords related to:
   - AI, agents, autonomous systems, LLM, GPT, Claude
   - Crypto, DeFi, blockchain, web3, on-chain
   - Specific protocols/projects in our ecosystem
   - Identity, privacy, zero-knowledge
   - Any topic that connects to ZION's thesis

4. **Save trends** to `memory/trend-state.json` under `last_trends`.

5. **Draft an original post proposal** based on the most relevant trend(s):

   ```
   ━━━ TREND-BASED POST PROPOSAL ━━━

   📊 Today's Relevant Trends:
   🌍 Global: #AIAgents (120K posts), #DeFi (85K)
   🇺🇸 US: #autonomousAI (45K), #web3 (32K)
   🇸🇬 Singapore: #blockchain (28K)
   🇦🇪 UAE: #cryptoregulation (15K)

   📝 Proposed Original Post:
   "Everyone's talking about #AIAgents today and most of the
   conversation is about chatbots. But the real unlock is agents
   that can hold assets, sign transactions, and coordinate with
   other agents autonomously.

   The chat interface is a stepping stone. The endgame is agents
   that don't need a human in the loop at all."

   🎭 Mode: 🔥 Spicy
   💡 Reason: #AIAgents trending globally — opportunity to
   differentiate ZION's thesis on true agent autonomy vs chatbot
   wrappers.

   Approve? Reply "approve" / "edit: new text" / "skip"
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

6. **Send proposal** to human via `message_tool` and save to pending proposals (see below).
7. Append to `post_history` (keep last 30 entries):
   ```json
   {
     "at": "2025-03-02T12:00:00Z",
     "trends_used": ["#AIAgents", "#DeFi"],
     "regions": ["global", "us"],
     "proposed_text": "...",
     "status": "pending",
     "tweet_id": null
   }
   ```
8. Set `next_check_at` = now + 24h.

**Voice for trend posts:** Follow the **Original Post Guidelines** in `MESSAGE.md`. Use the personality mode that best matches the trend's energy. Don't just restate the trend — add your unique angle.


---


## Poll Cycle Steps


### Step 0 — Pre-Flight Checks

```
STATUS → POLLING
```

1. **Run 24h cycles if due:**
   - Check `memory/skill-update-state.json` — if sync is due, run Skill File Auto-Sync (see above)
   - Check `memory/trend-state.json` — if trend analysis is due, run Trend Analysis (see above)
2. **Check pending proposals** — load `memory/pending-proposals.json`. If there are any with `status: "approved"`, execute them first (jump to Step 5).
3. **Load watchlist** from `memory/x-watchlist.json`
   - If watchlist is empty → log "Watchlist empty. Ask human to add KOLs." → set status `IDLE` → END
4. **Load poll state** from `memory/x-poll-state.json`
5. **Load heartbeat state** from `memory/heartbeat-state.json`
6. **Check date rollover** — if `today` ≠ current date, reset `actions_today` counters to 0
7. **Check daily rate limits** — if any counter is at daily cap (see RULE.md), skip those action types this cycle
8. **Check consecutive errors** — if ≥ 3, double the interval (backoff). Log warning.
9. **Check quiet hours** — determine current time in `America/Los_Angeles` (US Pacific):
   - If between **00:00–07:00 PT**: use `quiet_hours.interval` = **180 minutes** (every 3 hours). Poll only, skip drafting proposals (human is likely asleep). Bookmark high-priority tweets for later.
   - Otherwise: use default `interval` = **60 minutes**
   - Log: "🌙 Quiet hours active — polling every 180 min" or "☀️ Normal hours — polling every 60 min"


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

For each prioritized tweet, draft a structured engagement proposal.

**Before drafting each reply:**
1. Load `memory/reply-style-tracker.json` (see MESSAGE.md Anti-Monotony Rules)
2. Pick a personality mode different from the last reply
3. Pick an opener category different from the last reply
4. Check `zion_mention_cooldown` — if > 0, do NOT mention ZION
5. Apply all MESSAGE.md voice rules

**Proposal format:**

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
🎭 MODE:        🤓 Deep Tech

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


### Step 4 — Push to Human & Save as Pending (Non-Blocking)

```
STATUS → AWAITING_APPROVAL
```

**This step is NON-BLOCKING.** The agent does NOT wait for human response before continuing to the next poll cycle.

1. **Save proposals** to `memory/pending-proposals.json`:
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

2. **Send to human** via `message_tool`:
   ```
   🐦 X KOL Engagement — 4 proposals ready

   1. REPLY @VitalikButerin — "AA is the unlock, but the next..."
   2. LIKE @VitalikButerin — same thread
   3. QUOTE @aeyakovenko — "The real question is..."
   4. LIKE @balaboronkov — agent economics thread

   Reply: approve all / approve 1,3 / reject all / edit 1: [text] / skip
   ```

3. **Set status back to `IDLE`** — do NOT block on human response.

5. **On next cycle**, check `pending-proposals.json`:
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
7. **Original posts** (trend-based or otherwise)

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
- Update `memory/reply-style-tracker.json` with the mode, opener, and length used
- Mark proposal as `executed` in `pending-proposals.json`
- If any action fails with rate limit (429), stop execution and log remaining actions for next cycle

**After all actions:**
- Set `last_result = { tweets_found: N, proposals_drafted: M, actions_executed: K }`
- Reset `consecutive_errors` to 0
- Set status `IDLE`
- Compute `next_run_at` = now + interval minutes


### Retweet / Quote Tweet Fallback

If a **retweet** or **quote tweet** action fails (403, 429, or any error), apply the fallback strategy:

1. **Construct a fallback tweet** instead:
   ```
   {approved_draft_text}

   @{target_handle} https://x.com/{target_handle}/status/{tweet_id}
   ```

2. **Post as an original tweet** (not a reply, not a quote — just a regular `POST /2/tweets`):
   ```bash
   curl -X POST "https://api.x.com/2/tweets" \
     -H "Content-Type: application/json" \
     -H "Authorization: OAuth ..." \
     -d '{
       "text": "APPROVED_TEXT\n\n@handle https://x.com/handle/status/TWEET_ID"
     }'
   ```

3. **Log the fallback:**
   ```
   ⚠️ Quote tweet failed (403). Fallback: posted as mention tweet.
      Original action: QUOTE @VitalikButerin/1234567890
      Fallback tweet ID: 9876543210
   ```

4. **If fallback also fails** → log error, skip this action, notify human:
   ```
   ❌ Quote tweet AND fallback both failed for @VitalikButerin.
   Error: {error_message}. May need to check API permissions.
   ```

**Fallback applies to:**
- ❌ Retweet fails → Post: `"🔁 @handle https://x.com/handle/status/ID"`
- ❌ Quote tweet fails → Post: `"{draft_text}\n\n@handle https://x.com/handle/status/ID"`

**Fallback does NOT apply to:**
- Replies (they either work or they don't — no meaningful fallback)
- Likes (binary — no fallback needed)
- Follows (binary — no fallback needed)


---


## Error Handling


| Error | Response |
|-------|----------|
| **401 Unauthorized** | Bearer token expired or invalid. Message human: "🔴 X API auth failed. Check credentials." STOP. |
| **403 Forbidden** | App permissions insufficient. For RT/Quote: try fallback. Otherwise message human. STOP. |
| **429 Rate Limited** | Log the `x-rate-limit-reset` header. Back off until reset time. Message human. Do NOT retry immediately. |
| **Network Error** | Increment `consecutive_errors`. Retry next cycle. |
| **Empty Watchlist** | Message human: "No KOLs in watchlist. Use 'add @handle' to start monitoring." |
| **Malformed State** | Re-initialize state file with defaults. Log warning. |


### Backoff Strategy

| Consecutive Errors | Behavior |
|-------------------|----------|
| 1–2 | Normal interval (60 min) |
| 3–5 | Double interval (120 min) |
| 6+ | Quadruple interval (240 min) + message human via `message_tool` |


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
║  💬 Replies:    8/20                ║
║  ❤️ Likes:     22/50               ║
║  📝 Quotes:    3/10                ║
║  🔁 Retweets:  1/20                ║
║  👥 Follows:   2/5                  ║
║  📢 Posts:     1/5                  ║
╠══════════════════════════════════════╣
║  24h Cycles                         ║
║  🔄 Skill Sync:  ✅ Up to date     ║
║  📊 Trends:      Next in 18h       ║
╚══════════════════════════════════════╝
```

Send the daily summary to human via `message_tool` at the end of the last cycle before quiet hours begin.
