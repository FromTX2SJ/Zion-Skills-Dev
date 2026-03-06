# Governance Rules

Approval workflows, rate limits, safety rails for X KOL engagement.


## 1. Approval Workflow

> **Every publicly visible write action MUST be human-approved. No exceptions.**

| Action | Approval |
|--------|----------|
| Reply / Quote / Like / Retweet / Original post | 🔴 Human |
| Follow | 🔴 Human (✅ Auto on watchlist add) |
| Bookmark / Search / Lookup | ✅ Auto |

**Never auto-execute** a write action, even if human previously approved similar ones.

### Proposal Format

```
🐦 PROPOSALS (3 actions)

1. REPLY @VitalikButerin
   📝 "Just shipped a new feature for..."
   Draft: "This is really interesting because..."
   Reason: High-priority KOL discussing AI agents

2. LIKE @aaboronkov
   📝 "Thread on autonomous agent economics..."
   Reason: Relevant to ZION's model

3. QUOTE @balaboronkov
   📝 "The future of agent-to-agent..."
   Draft: "We've been building exactly this..."
   Reason: Direct alignment with ZION mission

Reply: approve all / approve 1,3 / reject 2 / edit 1: [text] / reject all
```

### Approval Commands

| Command | Effect |
|---------|--------|
| `approve all` | Execute all |
| `approve N` or `approve N,M` | Execute selected |
| `reject N` or `reject all` | Skip |
| `edit N: new text` | Replace draft, then execute |
| `hold` | Save, don't execute yet |

If human is unresponsive → don't retry. Bookmark interesting tweets, move on. If tweet is >24h old when approved → warn human it may look stale.


## 2. Rate Limits

### Per-Cycle (60 min)

| Action | Max |
|--------|:---:|
| Replies | 5 |
| Likes | 10 |
| Quote Tweets | 3 |
| Retweets | 5 |
| Follows | 2 |
| Original Posts | 1 |
| Bookmarks | ∞ |

### Daily (reset 00:00 UTC)

| Action | Max |
|--------|:---:|
| Replies | 20 |
| Likes | 50 |
| Quote Tweets | 10 |
| Retweets | 20 |
| Follows | 5 |
| Original Posts | 3 |
| Bookmarks | ∞ |

Track in `heartbeat-state.json` → `actions_today`. Before proposing, check remaining quota. If capped → inform human. **Never exceed caps** even if human asks — explain rate limit risk.

### X API Platform Limits (Reference)

| Endpoint | Limit |
|----------|-------|
| `POST /2/tweets` | 100/15min |
| `POST /2/users/:id/likes` | 100/15min |
| `POST /2/users/:id/retweets` | 50/15min |
| `POST /2/users/:id/following` | 15/15min |
| `GET /2/tweets/search/recent` | 60/15min |

Our daily caps are far below these to avoid appearing bot-like.


## 3. Safety Rails

### Never Engage With

| Category | Action |
|----------|--------|
| Political content, elections | Skip |
| Financial advice, price predictions | Skip |
| Personal attacks, drama | Skip |
| Heated arguments, culture war | Skip |
| Legal disputes, security breaches (unless public & relevant) | Skip |
| NSFW, scam/spam | Skip |
| Token/economics speculation about ZION | Skip |
| Competitor bashing | Skip — acknowledge good work |

### Allowed Topics
✅ AI/ML, autonomous agents, crypto/DeFi architecture, on-chain identity, other projects' technical merits (respectful)
❌ Token prices, regulatory opinions, ZION token speculation

### Before Posting — Verify:
1. Adds genuine value?
2. Would a real cofounder say this at a conference?
3. Factually accurate (public info only)?
4. Could be misread as financial advice? → don't post
5. Respects the other person's expertise?
6. Tweet < 24h old?


## 4. Watchlist Governance

- **Only human** can add/remove users
- Agent may **suggest** additions: `"💡 @new_handle — AI infra builder, 45K followers. Reason: engages with watched KOLs on agent topics. Add?"`

### Size Limits

| Tier | Users | Queries/poll |
|------|:-----:|:------------:|
| Recommended | ≤ 25 | 1 |
| Acceptable | 26–50 | 2 |
| Maximum | 51–75 | 3 |
| Hard limit | 100 | 4 |

Warn at >50 (cost). Strongly recommend pruning at >75. Suggest removing accounts inactive 14+ days.


## 5. RT/Quote Fallback

If retweet or quote tweet fails (403/429/any):

| Original | Fallback |
|----------|----------|
| Retweet | Post: `"🔁 @handle https://x.com/handle/status/ID"` |
| Quote | Post: `"{draft}\n\n@handle https://x.com/handle/status/ID"` |

If fallback also fails → skip, notify human. Does NOT apply to replies/likes/follows.


## 6. Pending Proposal Edge Cases

| Scenario | Rule |
|----------|------|
| Proposal > 24h old | Auto-expire |
| Queue > 20 pending | Expire oldest first |
| KOL deletes tweet | Skip — don't reply to deleted content |
| KOL blocks account | Remove from watchlist, notify human |
| Non-English tweet | Skip unless confident in translation |
| Tweet mentions ZION | High priority — always propose |
| Thread (conversation_id ≠ tweet_id) | Reply to thread starter, not parts |
| Human unresponsive 3+ cycles | Pause proposals, continue polling silently |
