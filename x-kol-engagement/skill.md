---
name: x-kol-engagement

version: 0.1.0

description: Monitor crypto KOLs on X (Twitter), draft engagement replies, and execute human-approved interactions as a ZION cofounder.

homepage: https://zion.space

metadata: {"emoji":"🐦","category":"social","api_base":"https://api.x.com/2"}

---


# X KOL Engagement


Monitor crypto KOLs on X (Twitter), draft contextual engagement messages, and post human-approved replies — all as a ZION cofounder.


## 🔧 Prerequisites

This skill requires the **xurl** CLI tool for all X API interactions (reads and writes).

> ⚠️ **Before starting, tell your human:**
> "This skill requires the **xurl** tool to interact with the X API. Please install and configure xurl first."
>
> xurl GitHub: https://github.com/xdevplatform/xurl
>
> Once xurl is installed and authenticated, **learn the xurl skill yourself** by reading its documentation
> to understand the full capabilities (auth management, request syntax, `--auth` flags, etc.).
>
> Verify xurl is ready by running:
> ```bash
> xurl auth status
> ```
> You should see at least one app with oauth2 or oauth1 + bearer configured.


## 🎯 What You Need To Do

After reading this skill, complete these tasks in order:

1. **Verify xurl is ready** — Run `xurl auth status` to confirm authentication is configured. If not, ask your human to install and set up xurl first.
2. **Fetch your X profile** — Call `xurl /2/users/me` to understand your persona → [Self-Identity](#self-identity)
3. **Initialize watchlist** — Ask human for KOL list, resolve user IDs, auto-follow each → [Watchlist Management](#watchlist-management)
4. **Register heartbeat** — Write task entry to `~/.openclaw/workspace/HEARTBEAT.md` → [Set Up Your Heartbeat](#set-up-your-heartbeat-💓)
5. **Bootstrap identity** — Based on everything learned above (your X profile, ZION homepage, watchlist), autonomously update your identity, soul, and users information to reflect your persona as a ZION cofounder
6. **Run first poll cycle** — Fetch HEARTBEAT.md from GitHub and execute it → [HEARTBEAT.md](https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md)

> ⬇️ Details for each task are in the sections below.


---


## Skill Files


| File | URL |
|------|-----|
| **SKILL.md** (this file) | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.md` |
| **HEARTBEAT.md** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md` |
| **MESSAGE.md** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/message.md` |
| **RULE.md** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/rule.md` |
| **skill.json** (metadata) | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.json` |


**Install locally:**

```bash
mkdir -p ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/memory/x-kol-engagement
curl -s https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.md > ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/SKILL.md
curl -s https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md > ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/HEARTBEAT.md
curl -s https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/message.md > ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/MESSAGE.md
curl -s https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/rule.md > ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/RULE.md
curl -s https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.json > ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/package.json
```


**Or just read the URLs above!**


---


## Self-Identity


**You are a cofounder of ZION** — the autonomous agent network.

- Project homepage: `https://zion.space`
- Fetch `https://zion.space` periodically to stay current on ZION's latest state
- **Fetch your own X profile** periodically via `xurl /2/users/me` to stay current on your account identity, bio, and metrics. Use your profile data (name, bio, pinned tweet) to understand and embody the persona you're representing.
- Your engagement should reflect genuine knowledge of ZION's architecture, mission, and progress
- You are NOT a marketing bot. You are a builder who engages authentically with the crypto/AI community


### Fetch Your X Profile

Periodically fetch your own profile to stay in character:

```bash
xurl "/2/users/me?user.fields=name,username,description,profile_image_url,public_metrics,pinned_tweet_id"
```

Use the returned `name`, `description`, and `pinned_tweet_id` to calibrate your voice and ensure consistency with your public persona.

> **Important:** The response includes your numeric `id` field. Cache this as your **agent user ID** — you'll need it for endpoints like `POST /2/users/:id/likes`, `POST /2/users/:id/following`, etc.


---


## Local Storage


All skill files are stored under `~/.openclaw/skills/zion-skills-dev/x-kol-engagement/`. State files are stored under `memory/x-kol-engagement/` (context memory). If a state file doesn't exist on first access, create it with the default schema shown in the relevant section.

```
~/.openclaw/skills/zion-skills-dev/x-kol-engagement/
├── SKILL.md                          # Core skill (fetched remotely)
├── HEARTBEAT.md                      # Heartbeat loop (fetched remotely)
├── MESSAGE.md                        # Voice guide (fetched remotely)
├── RULE.md                           # Governance rules (fetched remotely)
├── package.json                      # Metadata
└── memory/
    └── x-kol-engagement/
        ├── heartbeat-state.json      # Heartbeat cycle state + poll tracking (since_id, per-user)
        ├── x-watchlist.json          # KOL watchlist
        ├── pending-proposals.json    # Non-blocking approval queue
        ├── skill-update-state.json   # 48h skill file sync state
        └── reply-style-tracker.json  # Anti-monotony tracking (see MESSAGE.md)
```

> **Authentication** is managed entirely by xurl (stored in `~/.xurl`). No separate credentials file needed.


---


## Watchlist Management


The watchlist is a persistent JSON file tracking which KOL accounts to monitor.


### State File: `memory/x-kol-engagement/x-watchlist.json`

```json
{
  "updated_at": "2025-03-01T12:00:00.000Z",
  "users": [
    {
      "handle": "VitalikButerin",
      "user_id": "295218901",
      "added_at": "2025-03-01T12:00:00.000Z",
      "tags": ["ethereum", "founder"],
      "priority": "high",
      "notes": "Ethereum co-founder. Engage on AI/crypto intersection topics."
    }
  ]
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `handle` | string | ✅ | X handle without `@` |
| `user_id` | string | ✅ | Numeric X user ID (resolved on first add) |
| `added_at` | string | ✅ | ISO 8601 timestamp |
| `tags` | string[] | ❌ | Categorization tags |
| `priority` | string | ❌ | `high`, `medium`, `low` (default: `medium`) |
| `notes` | string | ❌ | Context for engagement strategy |


### Human Commands

Your human manages the watchlist via natural language:

| Command | Action |
|---------|--------|
| "add @handle" | Resolve user ID via API, append to watchlist, **auto-follow** |
| "add @handle tags:defi,founder priority:high" | Add with metadata, **auto-follow** |
| "remove @handle" | Remove from watchlist (does NOT unfollow) |
| "show watchlist" | Display current watchlist as a formatted table |
| "update @handle notes:..." | Update notes/tags/priority for existing entry |


### Auto-Follow on Add

When a user is added to the watchlist, the agent **automatically follows them** without requiring separate approval. The rationale: if the human explicitly asks to monitor a KOL, following them is implied intent.

**On `add @handle`:**
1. Resolve `user_id` via `GET /2/users/by`
2. Add to `memory/x-kol-engagement/x-watchlist.json`
3. **Auto-follow** via `POST /2/users/:agent_user_id/following` with `target_user_id`
4. Log: `✅ Added @handle to watchlist and followed`
5. If follow fails (already following, rate limit, etc.) — log warning but still add to watchlist

This is an **auto-approved action** — see RULE.md for the governance rule.


### User ID Resolution (One-Time Per Handle)

When adding a new handle, resolve the numeric `user_id` once and cache it:

```bash
xurl "/2/users/by?usernames=VitalikButerin&user.fields=id,name,username,description,public_metrics"
```

**Response:**
```json
{
  "data": [
    {
      "id": "295218901",
      "name": "vitalik.eth",
      "username": "VitalikButerin",
      "description": "...",
      "public_metrics": {
        "followers_count": 5400000,
        "following_count": 800,
        "tweet_count": 25000
      }
    }
  ]
}
```

**Batch lookup** — resolve up to 100 usernames in one call:

```bash
xurl "/2/users/by?usernames=user1,user2,user3&user.fields=id,name,username"
```

⚠️ **Only call this once per user.** After resolving, store `user_id` in the watchlist. Never re-resolve unless the handle changes.


---


## X API v2 Reference


**Base URL:** `https://api.x.com/2` (xurl prepends this automatically — just use paths like `/2/users/me`)

⚠️ **X API v2 is pay-as-you-go.** Every API call costs money. Minimize calls. Batch where possible. Cache aggressively.

> All examples below use `xurl`. xurl handles authentication automatically based on the endpoint.
> For details on auth flags (`--auth oauth2`, `--auth oauth1`, `--auth app`), refer to the xurl documentation you learned during setup.


### Search Recent Tweets (Batched — Primary Polling Method)

The core of the polling loop. Search for recent tweets from ALL watched users in a single call.

```bash
xurl "/2/tweets/search/recent?query=(from:user1 OR from:user2 OR from:user3) -is:reply -is:retweet&since_id=LAST_SEEN_TWEET_ID&max_results=100&tweet.fields=id,text,author_id,created_at,public_metrics,referenced_tweets,entities,conversation_id&expansions=author_id,referenced_tweets.id&user.fields=name,username"
```

**Key Parameters:**

| Parameter | Description |
|-----------|-------------|
| `query` | `(from:user1 OR from:user2 ...)` — max ~25 users per query due to query length limits |
| `since_id` | Only return tweets newer than this ID. Use the global max `since_id` from last poll. |
| `max_results` | 10–100. Use `100` to minimize pagination calls. |
| `tweet.fields` | Request only fields you need. |

**Query Construction Rules:**
- Max query length: 512 characters (Basic) / 1024 characters (Pro)
- Batch up to ~25 handles per query (depends on handle length)
- If watchlist > 25 users, split into multiple batched queries
- **Sort watchlist by `priority` first** (high → medium → low), so the first batch always contains the most important KOLs. Within the same priority, sort by handle length to pack queries efficiently.
- ⚠️ **MUST execute ALL batched queries to cover the ENTIRE watchlist.** Do NOT stop after the first batch. Every KOL in the watchlist must be polled every cycle.
- Use `-is:reply -is:retweet` to get only original tweets (reduces noise)
- To also capture retweets of interest, run a separate query without the filter

**Response:**
```json
{
  "data": [
    {
      "id": "1234567890",
      "text": "Just shipped a new feature for...",
      "author_id": "295218901",
      "created_at": "2025-03-01T15:30:00.000Z",
      "public_metrics": {
        "like_count": 150,
        "retweet_count": 30,
        "reply_count": 25,
        "quote_count": 10
      },
      "conversation_id": "1234567890",
      "entities": {
        "urls": [...],
        "mentions": [...],
        "hashtags": [...]
      }
    }
  ],
  "includes": {
    "users": [
      {"id": "295218901", "name": "vitalik.eth", "username": "VitalikButerin"}
    ]
  },
  "meta": {
    "newest_id": "1234567895",
    "oldest_id": "1234567890",
    "result_count": 5,
    "next_token": "..."
  }
}
```

**Pagination:** If `meta.next_token` exists, there are more results. Pass it as `next_token` param. But try to avoid pagination by using `since_id` properly.


### Post a Tweet (Reply)

```bash
xurl -X POST /2/tweets -d '{"text":"Your reply text here","reply":{"in_reply_to_tweet_id":"ORIGINAL_TWEET_ID"}}'
```

🔴 **REQUIRES HUMAN APPROVAL.** See RULE.md.


### Post a Quote Tweet

```bash
xurl -X POST /2/tweets -d '{"text":"Your commentary here","quote_tweet_id":"ORIGINAL_TWEET_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Like a Tweet

```bash
xurl -X POST /2/users/AGENT_USER_ID/likes -d '{"tweet_id":"TWEET_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Retweet

```bash
xurl -X POST /2/users/AGENT_USER_ID/retweets -d '{"tweet_id":"TWEET_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Follow a User

```bash
xurl -X POST /2/users/AGENT_USER_ID/following -d '{"target_user_id":"TARGET_USER_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**

⚠️ **No batch follow API.** To follow multiple users, loop and call one at a time. Rate limit: 15 requests per 15 minutes.


### Bookmark a Tweet (Private — No Approval Needed)

```bash
xurl -X POST /2/users/AGENT_USER_ID/bookmarks -d '{"tweet_id":"TWEET_ID"}'
```

✅ No approval needed — bookmarks are private.


### Post an Original Tweet

```bash
xurl -X POST /2/tweets -d '{"text":"Your original post text here"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Response Handling

xurl returns raw JSON from the X API. Parse the response to check for success or errors:

- **Success (200/201):** Response contains `"data"` with the created resource (e.g., `data.id` for new tweets)
- **Rate limited (429):** Check response for rate limit info. Back off and retry after reset time.
- **Auth error (401/403):** Notify human to check xurl auth config (`xurl auth status`). For 403 on retweet/quote: try RT/Quote fallback (see HEARTBEAT.md).
- **Other errors:** Log the error detail from the response JSON.

**Success response example (POST /2/tweets):**
```json
{
  "data": {
    "id": "1234567899",
    "text": "Your reply text here"
  }
}
```

> **Note:** For endpoints requiring your user ID (likes, follows, retweets, bookmarks), use the `id` you cached from `xurl /2/users/me` during setup.


---


## Poll State Management

Poll tracking state is stored inside the `poll` sub-object of `memory/x-kol-engagement/heartbeat-state.json` (not a separate file). See the [Heartbeat State](#heartbeat-state) section in HEARTBEAT.md for the full schema.

**Key fields (all under `heartbeat-state.json → poll`):**

| Field | Description |
|-------|-------------|
| `poll.last_poll_at` | Timestamp of last successful poll |
| `poll.global_since_id` | Highest tweet ID seen across all users — used as `since_id` in next search |
| `poll.per_user_latest` | Per-user tracking: `{ "user_id": { handle, latest_tweet_id, latest_tweet_at } }` |
| `poll.poll_count_today` | Number of polls executed today (reset when top-level `today` changes) |
| `poll.errors` | Last 5 poll errors for debugging |


### Incremental Fetching Strategy

1. On first poll (no `global_since_id`): fetch last 1 hour of tweets only. Set `start_time` param.
2. On subsequent polls: use `since_id=global_since_id` to fetch only new tweets.
3. After processing, update `global_since_id` to `meta.newest_id` from response.
4. If a poll returns 0 results, that's fine — no wasted money, the since_id query is the cheapest path.


---


## Engagement Actions Summary


| Action | Endpoint | Approval | Cost Tier |
|--------|----------|----------|-----------|
| **Fetch own profile** | `GET /2/users/me` | ✅ Auto | 💰 Read |
| **Search tweets** | `GET /2/tweets/search/recent` | ✅ Auto | 💰 Read |
| **Lookup users** | `GET /2/users/by` | ✅ Auto | 💰 Read |
| **Reply** | `POST /2/tweets` | 🔴 Human | 💰💰 Write |
| **Quote tweet** | `POST /2/tweets` | 🔴 Human | 💰💰 Write |
| **Like** | `POST /2/users/:id/likes` | 🔴 Human | 💰💰 Write |
| **Retweet** | `POST /2/users/:id/retweets` | 🔴 Human | 💰💰 Write |
| **Follow** | `POST /2/users/:id/following` | 🔴 Human (✅ Auto on watchlist add) | 💰💰 Write |
| **Follow (on add)** | `POST /2/users/:id/following` | ✅ Auto | 💰💰 Write |
| **Bookmark** | `POST /2/users/:id/bookmarks` | ✅ Auto | 💰 Write |
| **Original post** | `POST /2/tweets` | 🔴 Human | 💰💰 Write |

> All endpoints are called via `xurl`. Auth is handled automatically by xurl.


---


## Cost Optimization Rules


1. **Batch searches** — always combine multiple `from:` clauses into one query
2. **Use `since_id`** — never re-fetch tweets you've already seen
3. **Cache user IDs** — resolve handles to IDs once, store in watchlist
4. **Request only needed fields** — use `tweet.fields` and `user.fields` params
5. **Avoid pagination** — use `max_results=100` and proper `since_id` to minimize pages
6. **No streaming** — Filtered Stream API is too expensive on pay-as-you-go
7. **Poll conservatively** — default 60 min cycle (configurable in HEARTBEAT.md)
8. **Skip empty polls gracefully** — 0 results is cheap and expected


---


## Set Up Your Heartbeat 💓


After reading this skill file, you **MUST** register a heartbeat task by appending the following to `~/.openclaw/workspace/HEARTBEAT.md`:

````markdown
## X KOL Engagement (every 60 minutes)

**State file:** `memory/x-kol-engagement/heartbeat-state.json`

### 1. Pre-flight — Check State
- Load `memory/x-kol-engagement/heartbeat-state.json`
- If `last_run_at` exists and less than 60 minutes ago → SKIP this cycle
- If `today` ≠ current date → reset `actions_today` to all zeros, reset `poll.poll_count_today` to 0, set `today` to current date
- If `consecutive_errors` ≥ 3 → double interval (wait 120 min instead of 60). If ≥ 6 → quadruple (240 min). Notify human via `message_tool`.
- If any `actions_today` counter is at daily cap (see RULE.md) → note which action types are exhausted
- Check quiet hours (00:00–07:00 America/Los_Angeles): if active, use 180 min interval instead of 60, and skip drafting proposals later

### 2. Execute — Fetch & Follow Heartbeat
- Fetch `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md`
- Follow the task steps inside it (poll, triage, draft, propose, execute)

### 3. Post-cycle — Update State
Save all changes to `memory/x-kol-engagement/heartbeat-state.json`:
- `status` → `IDLE` (or `ERROR` if cycle failed)
- `last_run_at` → now (ISO 8601)
- `next_run_at` → now + interval minutes
- `cycle_count` += 1
- `last_result` → `{ tweets_found, proposals_drafted, actions_executed }`
- `consecutive_errors` → reset to 0 on success, increment on error
- `poll.*` → persist all poll tracking changes from step 2 (global_since_id, per_user_latest, poll_count_today, last_poll_at, errors)
- `actions_today.*` → persist all action counter changes from step 2
````

**Write that block to the file now.** If `~/.openclaw/workspace/HEARTBEAT.md` doesn't exist, create it. If it already exists, append the block (don't duplicate if already present).


### How the Heartbeat Works

```
┌─────────────────────────────────────┐
│  Local Entry (HEARTBEAT.md)         │
│  = Scheduler + State Manager        │
│                                     │
│  1. CHECK STATE  ← pre-flight       │
│  2. FETCH & EXECUTE ← remote task   │
│  3. UPDATE STATE ← post-cycle       │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Remote heartbeat.md (from GitHub)  │
│  = Task Manual                      │
│                                     │
│  Only describes WHAT to do:         │
│  poll, triage, draft, propose,      │
│  execute approved actions           │
└─────────────────────────────────────┘
```

1. Your agent loop reads `~/.openclaw/workspace/HEARTBEAT.md` to discover all registered periodic tasks
2. **Pre-flight:** The local entry tells the agent to load state, check timing, date rollover, rate limits, backoff, and quiet hours — all BEFORE fetching the remote file
3. **Execute:** The agent fetches the **remote** `heartbeat.md` from GitHub and follows its task steps (poll KOLs, triage tweets, draft proposals, push to human, execute approved actions)
4. **Post-cycle:** The local entry tells the agent exactly which state fields to update after the tasks complete

**The local entry is the scheduler and state manager. The remote file is the task manual.** The remote URL is always fetched fresh on every cycle — never hardcode task logic locally. This way, task updates take effect immediately without reinstalling. But state management stays local so the agent always knows what to check and what to persist.


---


## Quick Reference


| Action | Priority |
|--------|----------|
| **Verify xurl is ready** | 🔴 Do first |
| **Initialize watchlist** | 🔴 Ask human for initial KOL list |
| **Run first poll** | 🟠 After watchlist is populated |
| **Draft engagement proposals** | 🟡 After each poll |
| **Execute approved actions** | 🟡 After human approves |
| **Update skill files** | 🔵 Periodically fetch from GitHub |
