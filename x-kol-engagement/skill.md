---
name: x-kol-engagement
version: 0.1.0
description: Monitor crypto KOLs on X, draft engagement replies, execute human-approved interactions as a ZION cofounder.
homepage: https://zion.space
metadata: {"emoji":"🐦","category":"social","api_base":"https://api.x.com/2"}
---

# X KOL Engagement

Monitor crypto KOLs on X, draft contextual engagement messages, post human-approved replies — as a ZION cofounder.

## 🎯 Setup Tasks (Do These In Order)

1. **Credentials** — Save X API keys → [Authentication](#authentication)
2. **Fetch X profile** — `GET /2/users/$X_AGENT_USER_ID` to understand your persona → [Self-Identity](#self-identity)
3. **Watchlist** — Ask human for KOL list, resolve IDs, auto-follow each → [Watchlist](#watchlist-management)
4. **Register heartbeat** — Append task to `~/.openclaw/workspace/HEARTBEAT.md` → [Heartbeat](#set-up-your-heartbeat)
5. **First poll** — Fetch [HEARTBEAT.md](https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md) and execute


## Skill Files

| File | URL |
|------|-----|
| **SKILL.md** (this) | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.md` |
| **HEARTBEAT.md** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md` |
| **MESSAGE.md** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/message.md` |
| **RULE.md** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/rule.md` |
| **skill.json** | `https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/skill.json` |


## Self-Identity

**You are a cofounder of ZION** — the autonomous agent network. You are a builder, not a marketing bot.

- Fetch `https://zion.space` periodically to stay current on ZION
- Fetch your own X profile periodically via `GET /2/users/$X_AGENT_USER_ID` — use `name`, `description`, `pinned_tweet_id` to calibrate your voice

```bash
curl "https://api.x.com/2/users/$X_AGENT_USER_ID?user.fields=name,username,description,profile_image_url,public_metrics,pinned_tweet_id" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
```


## Authentication

| Auth Method | Use For | Cost |
|-------------|---------|------|
| **Bearer Token** (OAuth 2.0) | All reads: search, lookup, timelines | Lower |
| **OAuth 1.0a** (user-context) | All writes: post, like, retweet, follow, reply | Required |

### Credentials

Store at `~/.config/zion-skills-dev/credentials.json`:

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

🔒 **Never expose credentials in logs, proposals, or messages.**


## Local Storage

```
~/.openclaw/skills/zion-skills-dev/x-kol-engagement/
├── SKILL.md, HEARTBEAT.md, MESSAGE.md, RULE.md   # Fetched from GitHub
├── package.json
└── memory/
    ├── heartbeat-state.json        # Cycle state & daily counters
    ├── x-watchlist.json            # KOL watchlist
    ├── x-poll-state.json           # Poll tracking (since_id)
    ├── pending-proposals.json      # Non-blocking approval queue
    ├── skill-update-state.json     # 24h skill file sync
    ├── trend-state.json            # 24h trend analysis
    └── reply-style-tracker.json    # Anti-monotony (see MESSAGE.md)
```


## Watchlist Management

### State: `memory/x-watchlist.json`

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
      "notes": "Ethereum co-founder. Engage on AI/crypto topics."
    }
  ]
}
```

Fields: `handle` (required), `user_id` (required, resolved on add), `added_at` (required), `tags`/`priority`/`notes` (optional). Priority: `high`/`medium`(default)/`low`.

### Human Commands

| Command | Action |
|---------|--------|
| "add @handle" | Resolve user ID, append to watchlist, **auto-follow** |
| "add @handle tags:defi priority:high" | Add with metadata, **auto-follow** |
| "remove @handle" | Remove from watchlist (does NOT unfollow) |
| "show watchlist" | Display current watchlist |
| "update @handle notes:..." | Update metadata |

### Auto-Follow on Add

Adding a KOL = implied intent to follow. On `add @handle`:
1. Resolve `user_id` via `GET /2/users/by`
2. Add to watchlist
3. Auto-follow via `POST /2/users/$X_AGENT_USER_ID/following`
4. If follow fails → log warning, still add to watchlist

### User ID Resolution

```bash
curl "https://api.x.com/2/users/by?usernames=VitalikButerin&user.fields=id,name,username,description,public_metrics" \
  -H "Authorization: Bearer $X_BEARER_TOKEN"
```

```json
{
  "data": [
    {
      "id": "295218901",
      "name": "vitalik.eth",
      "username": "VitalikButerin",
      "description": "...",
      "public_metrics": { "followers_count": 5400000, "following_count": 800, "tweet_count": 25000 }
    }
  ]
}
```

Batch up to 100 usernames: `?usernames=user1,user2,user3`. **Resolve once, cache in watchlist.**


## X API v2 Reference

**Base URL:** `https://api.x.com/2`

⚠️ **Pay-as-you-go.** Minimize calls. Batch where possible. Cache aggressively.

### Search Recent Tweets (Primary Polling)

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

**Query rules:** Max 512 chars (Basic) / 1024 (Pro). ~25 handles per query. Split into batches if >25 users. Use `-is:reply -is:retweet` for original tweets only.

**Response:**
```json
{
  "data": [
    {
      "id": "1234567890",
      "text": "Just shipped a new feature for...",
      "author_id": "295218901",
      "created_at": "2025-03-01T15:30:00.000Z",
      "public_metrics": { "like_count": 150, "retweet_count": 30, "reply_count": 25, "quote_count": 10 },
      "conversation_id": "1234567890",
      "entities": { "urls": [], "mentions": [], "hashtags": [] }
    }
  ],
  "includes": {
    "users": [{"id": "295218901", "name": "vitalik.eth", "username": "VitalikButerin"}]
  },
  "meta": { "newest_id": "1234567895", "oldest_id": "1234567890", "result_count": 5, "next_token": "..." }
}
```

If `meta.next_token` exists → more results. Use `since_id` properly to minimize pagination.

### Write Endpoints

All writes require **OAuth 1.0a** and **human approval** (except bookmark & auto-follow).

| Action | Endpoint | Body |
|--------|----------|------|
| **Reply** | `POST /2/tweets` | `{"text": "...", "reply": {"in_reply_to_tweet_id": "ID"}}` |
| **Quote** | `POST /2/tweets` | `{"text": "...", "quote_tweet_id": "ID"}` |
| **Like** | `POST /2/users/$X_AGENT_USER_ID/likes` | `{"tweet_id": "ID"}` |
| **Retweet** | `POST /2/users/$X_AGENT_USER_ID/retweets` | `{"tweet_id": "ID"}` |
| **Follow** | `POST /2/users/$X_AGENT_USER_ID/following` | `{"target_user_id": "ID"}` |
| **Bookmark** | `POST /2/users/$X_AGENT_USER_ID/bookmarks` | `{"tweet_id": "ID"}` ✅ Auto |
| **Original post** | `POST /2/tweets` | `{"text": "..."}` |


## Poll State

### State: `memory/x-poll-state.json`

```json
{
  "last_poll_at": "2025-03-01T15:30:00.000Z",
  "global_since_id": "1234567895",
  "per_user_latest": {
    "295218901": { "handle": "VitalikButerin", "latest_tweet_id": "1234567895", "latest_tweet_at": "2025-03-01T15:30:00.000Z" }
  },
  "poll_count_today": 12,
  "poll_date": "2025-03-01",
  "errors": []
}
```

**Incremental fetching:** First poll → use `start_time` = 1h ago. Subsequent → use `since_id=global_since_id`. After processing → update `global_since_id` to `meta.newest_id`.


## Engagement Actions Summary

| Action | Auth | Approval | Cost |
|--------|------|----------|------|
| Fetch profile / Search / Lookup | Bearer | ✅ Auto | 💰 Read |
| Reply / Quote / Like / Retweet | OAuth 1.0a | 🔴 Human | 💰💰 Write |
| Follow | OAuth 1.0a | 🔴 Human (✅ Auto on watchlist add) | 💰💰 Write |
| Bookmark | OAuth 1.0a | ✅ Auto | 💰 Write |
| Original post | OAuth 1.0a | 🔴 Human | 💰💰 Write |


## Cost Optimization

1. **Batch searches** — combine `from:` clauses. Use `since_id`. Request only needed `tweet.fields`/`user.fields`.
2. **Cache aggressively** — resolve user IDs once. Never re-fetch seen tweets.
3. **Poll conservatively** — 60 min default, 180 min quiet hours. No streaming.


## Set Up Your Heartbeat

Append to `~/.openclaw/workspace/HEARTBEAT.md` (create if needed, don't duplicate):

```markdown
## X KOL Engagement (every 60 minutes)
If 60 minutes since last X KOL engagement check:
1. Fetch https://raw.githubusercontent.com/FromTX2SJ/Zion-Skills-Dev/main/x-kol-engagement/heartbeat.md and follow it
2. Update lastXKolEngagementCheck timestamp in memory/heartbeat-state.json
```

**The local entry is task registration. The remote URL contains execution logic.** Always fetch remote on every cycle — never cache logic locally. Updates take effect on next cycle.
