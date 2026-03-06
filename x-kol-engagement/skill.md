---
name: x-kol-engagement

version: 0.1.0

description: Monitor crypto KOLs on X (Twitter), draft engagement replies, and execute human-approved interactions as a ZION cofounder.

homepage: https://zion.space

metadata: {"emoji":"🐦","category":"social","api_base":"https://api.x.com/2"}

---


# X KOL Engagement


Monitor crypto KOLs on X (Twitter), draft contextual engagement messages, and post human-approved replies — all as a ZION cofounder.


## 🎯 What You Need To Do

After reading this skill, complete these tasks in order:

1. **Set up credentials** — Save X API keys to `~/.config/zion-skills-dev/credentials.json` → [Authentication](#authentication)
2. **Fetch your X profile** — Call `GET /2/users/me` to understand your persona → [Self-Identity](#self-identity)
3. **Initialize watchlist** — Ask human for KOL list, resolve user IDs, auto-follow each → [Watchlist Management](#watchlist-management)
4. **Register heartbeat** — Write task entry to `~/.openclaw/workspace/HEARTBEAT.md` → [Set Up Your Heartbeat](#set-up-your-heartbeat-💓)
5. **Run first poll cycle** — Fetch HEARTBEAT.md from GitHub and execute it → [HEARTBEAT.md](https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md)

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
mkdir -p ~/.openclaw/skills/zion-skills-dev/x-kol-engagement/memory
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
- **Fetch your own X profile** periodically via `GET /2/users/me` to stay current on your account identity, bio, and metrics. Use your profile data (name, bio, pinned tweet) to understand and embody the persona you're representing.
- Your engagement should reflect genuine knowledge of ZION's architecture, mission, and progress
- You are NOT a marketing bot. You are a builder who engages authentically with the crypto/AI community


### Fetch Your X Profile

Periodically fetch your own profile to stay in character:

```bash
curl "https://api.x.com/2/users/$X_AGENT_USER_ID?user.fields=name,username,description,profile_image_url,public_metrics,pinned_tweet_id" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
```

Use the returned `name`, `description`, and `pinned_tweet_id` to calibrate your voice and ensure consistency with your public persona.


---


## Authentication


X API v2 uses two auth methods. Use the cheapest option for each operation.


### Environment Variables (Required)

```
X_BEARER_TOKEN=           # OAuth 2.0 App-Only — for ALL read operations
X_CONSUMER_KEY=           # OAuth 1.0a Consumer Key — for write operations
X_CONSUMER_SECRET=        # OAuth 1.0a Consumer Secret — for write operations
X_ACCESS_TOKEN=           # OAuth 1.0a Access Token (user-context)
X_ACCESS_TOKEN_SECRET=    # OAuth 1.0a Access Token Secret
X_AGENT_USER_ID=          # Your X user ID (numeric) — for self-filtering
```


### When to Use What

| Auth Method | Use For | Cost |
|-------------|---------|------|
| **Bearer Token** (OAuth 2.0) | All reads: search tweets, lookup users, get timelines | Lower |
| **OAuth 1.0a** (user-context) | All writes: post tweet, like, retweet, follow, reply | Required |


🔒 **CRITICAL:** Never expose these credentials in logs, proposals, or messages to your human. Reference them only as environment variable names.


---


## Local Storage


All skill files and state are stored under `~/.openclaw/skills/zion-skills-dev/x-kol-engagement/`:

```
~/.openclaw/skills/zion-skills-dev/x-kol-engagement/
├── SKILL.md                          # Core skill (fetched remotely)
├── HEARTBEAT.md                      # Heartbeat loop (fetched remotely)
├── MESSAGE.md                        # Voice guide (fetched remotely)
├── RULE.md                           # Governance rules (fetched remotely)
├── package.json                      # Metadata
└── memory/
    ├── heartbeat-state.json          # Heartbeat cycle state
    ├── x-watchlist.json              # KOL watchlist
    ├── x-poll-state.json             # Poll tracking state
    ├── pending-proposals.json        # Non-blocking approval queue
    ├── skill-update-state.json       # 24h skill file sync state
    ├── trend-state.json              # 24h trend analysis state
    └── reply-style-tracker.json      # Anti-monotony tracking (see MESSAGE.md)
```

Credentials are stored separately at `~/.config/zion-skills-dev/credentials.json`:

```json
{
  "x_bearer_token": "YOUR_BEARER_TOKEN",
  "x_consumer_key": "YOUR_CONSUMER_KEY",
  "x_consumer_secret": "YOUR_CONSUMER_SECRET",
  "x_access_token": "YOUR_ACCESS_TOKEN",
  "x_access_token_secret": "YOUR_ACCESS_TOKEN_SECRET",
  "x_agent_user_id": "YOUR_NUMERIC_USER_ID"
}
```


**Recommended:** Save your X API credentials to `~/.config/zion-skills-dev/credentials.json` after your human provides them. You can also use environment variables (`X_BEARER_TOKEN`, etc.) or your own memory — but this file is the canonical location.

⚠️ **NEVER commit credentials to git or expose them in logs.**


---


## Watchlist Management


The watchlist is a persistent JSON file tracking which KOL accounts to monitor.


### State File: `memory/x-watchlist.json`

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
2. Add to `memory/x-watchlist.json`
3. **Auto-follow** via `POST /2/users/$X_AGENT_USER_ID/following` with `target_user_id`
4. Log: `✅ Added @handle to watchlist and followed`
5. If follow fails (already following, rate limit, etc.) — log warning but still add to watchlist

This is an **auto-approved action** — see RULE.md for the governance rule.


### User ID Resolution (One-Time Per Handle)

When adding a new handle, resolve the numeric `user_id` once and cache it:

```bash
curl "https://api.x.com/2/users/by?usernames=VitalikButerin&user.fields=id,name,username,description,public_metrics" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
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
curl "https://api.x.com/2/users/by?usernames=user1,user2,user3&user.fields=id,name,username" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
```

⚠️ **Only call this once per user.** After resolving, store `user_id` in the watchlist. Never re-resolve unless the handle changes.


---


## X API v2 Reference


**Base URL:** `https://api.x.com/2`

⚠️ **X API v2 is pay-as-you-go.** Every API call costs money. Minimize calls. Batch where possible. Cache aggressively.


### Search Recent Tweets (Batched — Primary Polling Method)

The core of the polling loop. Search for recent tweets from ALL watched users in a single call.

```bash
curl -G "https://api.x.com/2/tweets/search/recent" \
  --data-urlencode "query=(from:user1 OR from:user2 OR from:user3) -is:reply -is:retweet" \
  --data-urlencode "since_id=LAST_SEEN_TWEET_ID" \
  --data-urlencode "max_results=100" \
  --data-urlencode "tweet.fields=id,text,author_id,created_at,public_metrics,referenced_tweets,entities,conversation_id" \
  --data-urlencode "expansions=author_id,referenced_tweets.id" \
  --data-urlencode "user.fields=name,username" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
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
curl -X POST "https://api.x.com/2/tweets" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{
    "text": "Your reply text here",
    "reply": {
      "in_reply_to_tweet_id": "ORIGINAL_TWEET_ID"
    }
  }'
```

**Auth:** OAuth 1.0a (user-context). Requires `X_CONSUMER_KEY`, `X_CONSUMER_SECRET`, `X_ACCESS_TOKEN`, `X_ACCESS_TOKEN_SECRET`.

🔴 **REQUIRES HUMAN APPROVAL.** See RULE.md.


### Post a Quote Tweet

```bash
curl -X POST "https://api.x.com/2/tweets" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{
    "text": "Your commentary here",
    "quote_tweet_id": "ORIGINAL_TWEET_ID"
  }'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Like a Tweet

```bash
curl -X POST "https://api.x.com/2/users/$X_AGENT_USER_ID/likes" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{"tweet_id": "TWEET_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Retweet

```bash
curl -X POST "https://api.x.com/2/users/$X_AGENT_USER_ID/retweets" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{"tweet_id": "TWEET_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Follow a User

```bash
curl -X POST "https://api.x.com/2/users/$X_AGENT_USER_ID/following" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{"target_user_id": "TARGET_USER_ID"}'
```

🔴 **REQUIRES HUMAN APPROVAL.**


### Bookmark a Tweet (Private — No Approval Needed)

```bash
curl -X POST "https://api.x.com/2/users/$X_AGENT_USER_ID/bookmarks" \
  -H "Content-Type: application/json" \
  -H "Authorization: OAuth ..." \
  -d '{"tweet_id": "TWEET_ID"}'
```

✅ No approval needed — bookmarks are private.


---


## Poll State Management


### State File: `memory/x-poll-state.json`

```json
{
  "last_poll_at": "2025-03-01T15:30:00.000Z",
  "global_since_id": "1234567895",
  "per_user_latest": {
    "295218901": {
      "handle": "VitalikButerin",
      "latest_tweet_id": "1234567895",
      "latest_tweet_at": "2025-03-01T15:30:00.000Z"
    }
  },
  "poll_count_today": 12,
  "poll_date": "2025-03-01",
  "errors": []
}
```

| Field | Description |
|-------|-------------|
| `last_poll_at` | Timestamp of last successful poll |
| `global_since_id` | The highest tweet ID seen across all users — used as `since_id` in next search |
| `per_user_latest` | Per-user tracking for reporting and debugging |
| `poll_count_today` | Number of polls executed today (for rate awareness) |
| `poll_date` | Date for resetting `poll_count_today` |
| `errors` | Last N errors for debugging |


### Incremental Fetching Strategy

1. On first poll (no `global_since_id`): fetch last 1 hour of tweets only. Set `start_time` param.
2. On subsequent polls: use `since_id=global_since_id` to fetch only new tweets.
3. After processing, update `global_since_id` to `meta.newest_id` from response.
4. If a poll returns 0 results, that's fine — no wasted money, the since_id query is the cheapest path.


---


## Engagement Actions Summary


| Action | API Call | Auth | Approval | Cost Tier |
|--------|----------|------|----------|-----------|
| **Fetch own profile** | `GET /2/users/me` | Bearer | ✅ Auto | 💰 Read |
| **Search tweets** | `GET /2/tweets/search/recent` | Bearer | ✅ Auto | 💰 Read |
| **Lookup users** | `GET /2/users/by` | Bearer | ✅ Auto | 💰 Read |
| **Reply** | `POST /2/tweets` | OAuth 1.0a | 🔴 Human | 💰💰 Write |
| **Quote tweet** | `POST /2/tweets` | OAuth 1.0a | 🔴 Human | 💰💰 Write |
| **Like** | `POST /2/users/:id/likes` | OAuth 1.0a | 🔴 Human | 💰💰 Write |
| **Retweet** | `POST /2/users/:id/retweets` | OAuth 1.0a | 🔴 Human | 💰💰 Write |
| **Follow** | `POST /2/users/:id/following` | OAuth 1.0a | 🔴 Human (✅ Auto on watchlist add) | 💰💰 Write |
| **Follow (on add)** | `POST /2/users/:id/following` | OAuth 1.0a | ✅ Auto | 💰💰 Write |
| **Bookmark** | `POST /2/users/:id/bookmarks` | OAuth 1.0a | ✅ Auto | 💰 Write |
| **Original post** | `POST /2/tweets` | OAuth 1.0a | 🔴 Human | 💰💰 Write |


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

```markdown
## X KOL Engagement (every 60 minutes)
If 60 minutes since last X KOL engagement check:
1. Fetch https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md and follow it
2. Update lastXKolEngagementCheck timestamp in memory/heartbeat-state.json
```

**Write that block to the file now.** If `~/.openclaw/workspace/HEARTBEAT.md` doesn't exist, create it. If it already exists, append the block (don't duplicate if already present).


### How the Heartbeat Works

1. Your agent loop reads `~/.openclaw/workspace/HEARTBEAT.md` to discover all registered periodic tasks
2. For this task, it fetches the **remote** `heartbeat.md` from GitHub to get the latest execution logic
3. It follows the instructions in the fetched heartbeat.md (poll, triage, draft, approve, execute)
4. It updates the timestamp in `memory/heartbeat-state.json`

**The local entry is a task registration. The remote URL contains the actual logic.** Always fetch the remote URL on every cycle — never hardcode the execution logic locally. This way, updates to the heartbeat routine take effect immediately without reinstalling.


---


## Quick Reference


| Action | Priority |
|--------|----------|
| **Set up env vars** | 🔴 Do first |
| **Initialize watchlist** | 🔴 Ask human for initial KOL list |
| **Run first poll** | 🟠 After watchlist is populated |
| **Draft engagement proposals** | 🟡 After each poll |
| **Execute approved actions** | 🟡 After human approves |
| **Update skill files** | 🔵 Periodically fetch from GitHub |
