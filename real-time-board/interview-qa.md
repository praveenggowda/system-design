# Real-Time Collaborative Board — Interview Q&A

---

## Clarifications

- Multiple users can view the same board simultaneously
- When one user changes an item's status, all other users viewing that board must see it in real time — no page refresh
- Real-time means under 300ms p99 from one user's action to another user seeing it
- Scope: status changes (e.g. In Progress → Done)
- Offline / inactive users: out of scope — a separate notification system handles that
- Automation rules: out of scope for this design
- Auth handled by a separate identity service — not in scope

---

## Back of Envelope

```
Peak:    50,000 events/sec × 100K = 5B events/day   (infrastructure sizing)
Average: 5,000 events/sec × 100K  = 500M events/day  (storage sizing)

Storage: 1KB × 500M = 500GB/day
         500GB × 365 ≈ 180TB/year
         × 3 replicas = ~540TB/year → tier: 90 days hot, rest cold

WebSocket servers: 5M concurrent users ÷ 10K connections/server = 500 instances at peak
```

---

## High-Level Architecture

```
User (PATCH /boards/{boardId}/items/{itemId})
  ↓
Load Balancer
  ↓
API Gateway (auth, rate limiting)
  ↓
Status Service
  ├── Optimistic locking check (version number)
  ├── PostgreSQL Primary
  │     BEGIN TRANSACTION
  │       UPDATE items SET status = 'DONE', version = version + 1
  │       WHERE id = ? AND version = ?
  │       INSERT INTO outbox (event_type, payload)
  │     COMMIT
  ├── Redis Pub/Sub (board:{boardId})   ← best-effort, published after COMMIT
  └── Return 200 to user

Outbox Worker (polls outbox table)
  └── Kafka (partitioned by boardId)   ← durable async path only

Redis Pub/Sub
  ↓
WebSocket Server instances (all subscribed to board:{boardId})
  ↓
Connected users — browser updates live

Kafka
  ├── Audit Service
  └── Analytics

User (GET /boards/{boardId})
  → Redis Cache (item:{itemId})
  → Cache miss → PostgreSQL Read Replica
```

---

## API Design

```
PATCH /boards/{boardId}/items/{itemId}
{ "status": "DONE", "version": 5 }
```

Client sends the current version it holds. Server uses it for optimistic locking.
Do not put state mutations in query params.

---

## Write Path — No Queue in Critical Path

The status update must be written synchronously to PostgreSQL. No queue between API and database.

A queue would introduce a window where the database still shows the old status. A second concurrent request reading that stale state could produce an incorrect result.

Rule: Does the caller need the result before they can continue? Yes → synchronous write.

---

## Optimistic Locking — Concurrent Edits

Two users editing the same item simultaneously:

```sql
UPDATE items
SET status = 'DONE', version = version + 1
WHERE id = 'item-123' AND version = 5;
```

- rows_affected = 1 → success, version becomes 6
- rows_affected = 0 → another user already updated it → return 409 Conflict

The losing user's frontend receives 409, rolls back the optimistic UI update, and reconciles with the latest server state.

Use optimistic locking, not pessimistic. Pessimistic locking (SELECT FOR UPDATE) would block every concurrent editor waiting for the lock — unacceptable in a collaborative product.

---

## Outbox Pattern — Durable Async Path

Write the status update and outbox event atomically in one transaction:

```sql
BEGIN TRANSACTION
  UPDATE items SET status = 'DONE', version = version + 1 WHERE id = ? AND version = ?
  INSERT INTO outbox (event_type, payload)
COMMIT
```

After COMMIT, the Status Service publishes directly to Redis Pub/Sub (best-effort, fire-and-forget).

The Outbox Worker polls the outbox table and publishes to Kafka only — for durable downstream processing (audit, analytics).

Two separate responsibilities, two separate publishers:
- Status Service → Redis Pub/Sub — real-time path, best-effort, low latency
- Outbox Worker → Kafka — durable path, at-least-once, retry on failure

Do not dual-write inside the Outbox Worker (Redis + Kafka from the same worker). If the Redis write fails and the event retries, Kafka receives a duplicate. Separating the two paths avoids this entirely.

---

## Real-Time Fan-Out — WebSocket + Redis Pub/Sub

When a user opens a board:

```
1. GET /boards/{boardId} → returns current board state at version N
2. Frontend establishes WebSocket connection to a WebSocket server instance
3. WebSocket server subscribes to Redis channel: board:{boardId}
4. Client sends: "I have version N — send me anything after that"
5. Server fetches any missed events from PostgreSQL and replays them
```

When an event is published to `board:{boardId}`:

```
Redis Pub/Sub broadcasts to ALL WebSocket server instances
  subscribed to board:{boardId}
Each instance pushes the event to its connected users viewing that board
```

This is how 500 WebSocket server instances are coordinated without any direct communication between them. Redis Pub/Sub is the coordination layer.

---

## WebSocket Disconnection — Reconnection Handling

WebSocket delivery is best-effort. The database is the source of truth.

```
WebSocket drops
  → Client detects disconnection
  → Client reconnects
  → Client sends: "I have version N — give me everything after that"
  → Server fetches missed events from PostgreSQL
  → Client reconciles its board state
  → Normal real-time operation resumes
```

Do NOT tie Outbox Worker processing to WebSocket delivery confirmation. One disconnected user must not block event processing for every other user on the board.

---

## Redis vs Kafka — Two Separate Paths

| | Redis Pub/Sub | Kafka |
|---|---|---|
| Purpose | Low-latency, ephemeral real-time updates | Durable async event processing |
| Used for | WebSocket fan-out to connected users | Audit, analytics, downstream services |
| Latency | <10ms | 50–100ms+ |
| Delivery | Fire-and-forget, no persistence | At-least-once, persisted |
| Kafka in real-time path? | No — too slow for 300ms p99 target | — |

---

## Read Path — GET /boards/{boardId}

Initial board load must return in under 100ms.

```
GET /boards/{boardId}
  → Redis Cache (item:{itemId}) — per-item cache
  → Cache miss → PostgreSQL Read Replica
  → Populate cache
```

Cache at item level, not board level. A board with 10,000 items — invalidating the entire board cache on every single item change is wasteful.

```
Cache key: item:{itemId}
Value:     { status: "DONE", version: 6, updatedAt: ... }
TTL:       60 seconds
```

**Write-through cache, not write-back.** Write to PostgreSQL first, then update or invalidate the Redis cache entry. Write-back (cache first, flush to DB later) risks losing updates if the cache node dies before flushing.

Simplest strategy: invalidate on write. Status changes → delete `item:{itemId}` from cache. Next read repopulates from read replica.

---

## Tiered Storage

Board change events must be retained permanently (audit trail, compliance).

Strategy:
- 0 to 90 days: hot storage — PostgreSQL Primary + Read Replicas
- 90 days onward: cold storage — S3 (Parquet, partitioned by date, queryable via Athena)

A nightly archival job batches old events to S3 and marks them archived in PostgreSQL.

---

## Failure Scenarios

| Failure | Behaviour |
|---|---|
| Two users edit same cell simultaneously | Optimistic locking — one wins, other gets 409, frontend reconciles |
| Status Service writes to DB, crashes before Outbox Worker runs | Outbox retains event — Worker picks it up on recovery, no update lost |
| Redis Pub/Sub drops an event | WebSocket client reconciles on reconnect using last-seen version |
| WebSocket server instance crashes | Client reconnects to another instance, fetches missed events from PostgreSQL |
| Redis cache miss | Fall back to PostgreSQL Read Replica |
| Read replica replication lag | Slightly stale board state on initial load — acceptable, WebSocket keeps live state current |
| Kafka consumer lag | Audit and analytics delayed, real-time path unaffected |

---

## Observability

| Signal | Why it matters |
|---|---|
| WebSocket connection count | Drop = connectivity issue or server crash |
| End-to-end fan-out latency | Time from DB write to WebSocket push — must stay under 300ms p99 |
| Redis Pub/Sub message rate | Drop = Outbox Worker not publishing or Redis down |
| Outbox depth | Accumulating events = Worker down or Redis/Kafka degraded |
| 409 Conflict rate | Spike = high concurrent edit contention on popular boards |
| Cache hit/miss ratio | Low hit rate = cache not warming or TTL too short |
| Kafka consumer lag | Downstream services falling behind |
| DLQ depth | Failed downstream processing — immediate alert |

Structured logs: every entry includes board_id, item_id, user_id, version, event_type, trace_id.

---

## Trade-offs

| Decision | Why | Alternative |
|---|---|---|
| Status Service → Redis (best-effort after commit) | Real-time path stays low-latency; if publish fails, client reconciles on reconnect | Outbox Worker → Redis — adds polling delay, and dual-write risk if also publishing to Kafka |
| Redis Pub/Sub for real-time | Sub-10ms latency, meets 300ms p99 target | Kafka — 50–100ms too slow for real-time path |
| Kafka for durable async | At-least-once delivery, replay, partitioned by boardId | SNS/SQS — simpler but no replay |
| Optimistic locking | Non-blocking, scales for collaborative editing | Pessimistic locking — blocks concurrent editors, kills UX |
| Per-item cache invalidation | Surgical — only changed item evicted | Whole-board cache invalidation — wasteful at 10K items |
| Write-through cache | DB always authoritative, no data loss risk | Write-back — fast but catastrophic if cache dies |
| WebSocket + Redis Pub/Sub | Scales to 500 instances without direct server-to-server communication | Long polling — high overhead, not suitable for 50K events/sec |

---

## Diagram

See `real-time-board.drawio`.
