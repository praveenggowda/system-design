# Real-Time Balance — Mock Interview Q&A

## Clarification Phase

**Interviewer:** Design a system that allows customers to view their account balance in real time. The balance must always be accurate. Go.

**Praveen:** Before I start — a few clarifying questions.
- What account types are supported — savings, current, both?
- Are transactions stored as debit/credit entries or as a pre-computed balance?
- When you say real time — no eventual consistency? Balance must reflect the latest committed transaction?
- Consistency over availability — CP system?
- How many registered users and what is the DAU?
- What RPS are we expecting on the balance endpoint?
- What is the latency requirement?
- When a payment completes — does the user's next balance read reflect that payment immediately?

**Interviewer:** Savings and current — treat them the same. Transactions stored as debit/credit entries. Correct — no eventual consistency. CP. 10M registered users, 1M DAU. 5,000 RPS peak. 200ms p99. Yes — if a payment completes at 10:00:00, a balance read at 10:00:01 must reflect it.

**Praveen:** One more — does this service own the ledger or does it read from an upstream ledger service?

**Interviewer:** You do not own the ledger. Transactions are written by a separate Ledger Service. Your system receives transaction events and must serve accurate balances from them.

---

## Back of Envelope

- Average RPS: 500, Peak RPS: 5,000
- 500 * 86,400 = **43.2M requests/day**
- 43.2M / 1M DAU = 43.2 requests per user per day
- 1 balance response = ~1KB
- Peak throughput: 5,000 * 1KB = **5MB/sec**
- Storage: pre-computed balance table — 1 row per account, ~10M rows, negligible storage
- No sharding needed at this scale

---

## Non-Functional Requirements

- CP — consistency over availability. Balance must be accurate, not eventually consistent.
- 200ms p99 latency for GET /balance
- Zero stale reads — balance must reflect latest committed transaction
- Read-heavy workload — protect the Write Primary from read load

---

## High Level Design

**Praveen:** The core challenge is this: the balance must always reflect the latest committed transaction, but 5,000 RPS of reads cannot all hit the Write Primary.

My approach:
- The Ledger Service owns the transactions. When a transaction commits, it atomically updates a **pre-computed balance table** in the same database transaction — no summation on read, one row lookup.
- The Balance API serves GET /balance. It reads from **Redis cache** first. Cache hit returns immediately. Cache miss falls back to the Write Primary balance table, then writes the result back to Redis.
- The Ledger Service also publishes a `balance_updated` event to a **Redis pub/sub channel** after every transaction commit. The Balance API subscribes and updates Redis immediately — so the cache is always fresh.

**Interviewer:** Why not read from the Read Replica?

**Praveen:** Replication lag. Even a few milliseconds of lag means a user could read a balance that does not reflect their last transaction. The requirement says no eventual consistency — so I must read from the Write Primary as the fallback source of truth.

**Interviewer:** If every cache miss hits the Write Primary, how do you protect it?

**Praveen:** Redis is the primary protection. Most reads hit Redis. Cache misses are rare because the pub/sub mechanism keeps the cache fresh on every transaction commit. The Write Primary only gets hit when Redis has no entry for that account — first read after startup or after a Redis restart.

**Interviewer:** What if the Balance API is down for a moment and misses a pub/sub event?

**Praveen:** The cache entry becomes stale. When the next balance read comes in, it will hit Redis and get the old value — this violates the requirement. To handle this: cache entries have a very short TTL (1-2 seconds) as a safety net. If the pub/sub event was missed, the TTL expires quickly and the next read falls back to the Write Primary and repopulates Redis with the correct value.

---

## Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Read source | Write Primary (fallback) | Replication lag on Read Replica violates no-stale-read requirement |
| Cache invalidation | Redis pub/sub on every transaction commit | Cache is updated synchronously — no TTL-based staleness |
| Cache TTL | Very short (1-2 seconds) | Safety net for missed pub/sub events, not primary mechanism |
| Balance storage | Pre-computed balance table | One row lookup — no SUM() on every request |
| Balance update | Atomic with transaction commit | Same DB transaction — balance always consistent with ledger |
| Cache miss fallback | Write Primary → repopulate Redis | Source of truth, never stale |
