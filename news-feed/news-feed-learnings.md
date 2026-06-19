# News Feed System Design — Learnings

## What This Design Teaches

The news feed is one of the most important designs to master because it introduces:
- Fan-out pattern (write vs read)
- Pre-computed feeds with Redis
- Celebrity problem at scale
- Follower graph storage
- Async fan-out with SQS

---

## Core Problem

User follows 100 people. When they open the app they must see recent posts from all 100 people, merged and sorted by time. Must be fast — under 100ms.

---

## The Fan-Out Problem

When User A posts, 500 followers need to see it in their feed.

**Fan-out on Write** — push post to all followers' feeds immediately when post is created.
- Reads are instant — feed is pre-computed in Redis
- Problem: celebrity with 10 million followers causes massive write storm on every post

**Fan-out on Read** — when user opens app, pull posts from everyone they follow.
- No write storm
- Problem: user follows 1000 people — merging 1000 queries at read time is slow

**Hybrid approach — correct Senior answer:**

| User Type | Strategy |
|---|---|
| Normal users (< 10k followers) | Fan-out on Write — pre-compute feed in Redis |
| Celebrities (> 10k followers) | Fan-out on Read — fetch at read time and merge |

---

## Write Flow — User Creates Post

```
1. POST /CreatePost → API Gateway → Post Feed Service
2. Post Feed saves post to PostgreSQL Primary
3. Post Feed publishes to SQS
4. Fan-out Worker consumes from SQS
5. Fan-out Worker queries Followers Table — gets all follower IDs
6. Fan-out Worker writes post ID to each follower's Redis feed list
```

---

## Read Flow — User Opens App

```
1. GET /GetPosts → API Gateway → Read Feed Service
2. Read Feed fetches pre-computed feed from Redis (list of post IDs)
3. Read Feed fetches post details from PostgreSQL Read Replica
4. Returns merged and sorted feed
```

For celebrities — at step 2, also fetch latest posts from celebrities the user follows directly from DB and merge with Redis feed.

---

## Followers Table

Stores who follows who.

```
follower_id | following_id
user_A      | user_B        ← A follows B
user_A      | user_C        ← A follows C
```

When B posts, Fan-out Worker queries:
```sql
SELECT follower_id FROM followers WHERE following_id = 'user_B'
```

Gets all of B's followers and pushes post to their Redis feeds.

---

## Redis Feed Storage

Key: `feed:{userId}`
Value: sorted list of post IDs (most recent first)
TTL: 7 days

```
feed:user_A → [post_99, post_87, post_45, ...]
```

Read Feed fetches this list then batch-fetches post details from DB.

---

## SQS + DLQ

Post Feed → SQS (async, decoupled)
Fan-out Worker polls SQS
After 3 failed retries → message moves to DLQ automatically

DLQ preserves failed fan-out events for investigation and replay.

**Why SQS over Kafka here:**
- DLQ is built in natively — no custom implementation
- Fan-out is not ordered — SQS Standard is fine
- Simpler to operate

---

## Scale Numbers

- 10 million daily active users
- 1 million posts per day
- Average 500 followers per user
- 1 million posts × 500 followers = 500 million Redis writes per day
- Read heavy — 100:1 read to write ratio

---

## Topics Covered in This Design

| Topic | Concept |
|---|---|
| Fan-out pattern | Write vs Read vs Hybrid |
| Celebrity problem | Write storm — handle separately |
| Pre-computed feeds | Redis list per user |
| Follower graph | Separate table, queried on fan-out |
| Async messaging | SQS decouples Post Feed from Fan-out |
| Failure handling | DLQ for failed fan-out messages |
| Read scaling | Redis + Read Replica |
| Write path | PostgreSQL Primary |

---

## Key Interview Phrases

- "For normal users I use fan-out on write — posts are pushed to followers' Redis feeds immediately. For celebrities with millions of followers I use fan-out on read to avoid write storms"
- "The pre-computed feed in Redis is a list of post IDs per user — reads are sub-millisecond because no DB query is needed"
- "SQS decouples Post Feed from Fan-out Worker — if Fan-out is slow, posts still save successfully and fan-out catches up asynchronously"
- "DLQ is built into SQS natively — after 3 retries, failed messages move to DLQ automatically. No messages are lost"
- "The follower table is queried by the Fan-out Worker to get all followers of the poster — this is the fan-out list"
