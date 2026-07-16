# Item Reordering — monday.com E2E Design

---

## Clarifications

- 5,000 reorder events/second average, 20,000 at peak
- Up to 100,000 items per board, 50 concurrent users per board
- Two users reordering the same item simultaneously — one must win, no corruption
- Real-time update to all connected users required within milliseconds
- Optimistic UI: show update immediately, roll back on conflict

---

## Back of Envelope

| Metric | Value |
|---|---|
| Average throughput | 5,000 events/sec |
| Peak throughput | 20,000 events/sec |
| Daily volume (average) | 432M events/day |
| Per-event size | ~500 bytes |
| Daily storage (average) | ~216GB/day |
| 6-month retention | ~39TB |

Use peak (20,000/sec) for infrastructure sizing. Use average (5,000/sec) for storage budget.

---

## Ordering Strategy — Position Field

Use a `DECIMAL` position column from the start. Assign positions in large gaps:

```
Item A = 1000
Item B = 2000
Item C = 3000
```

Move Item C between A and B → calculate midpoint:

```
Item C = 1500
```

Only one row update. O(1).

Repeated insertions between the same two items will eventually exhaust numeric precision. At that point, **rebalance locally** — only the affected range or group, not the entire board.

Example local rebalance (group level):

```
Before: A=1000.00001  C=1000.00002  D=1000.00003  B=1000.00004
After:  A=1000        C=2000        D=3000        B=4000
```

Never rebalance 100,000 items at once. Scope to group or affected range to avoid contention.

---

## API Design — Client Sends Intent, Not Position

The client must NOT send a calculated position. The client may have stale board state.

Correct:

```
POST /boards/{boardId}/items/{itemId}/reorder
{
  "afterItemId": "A",
  "beforeItemId": "B",
  "version": 5
}
```

The server validates the current authoritative positions of A and B and calculates the new rank. This prevents stale client calculations causing incorrect ordering.

---

## Optimistic UI

The UI shows Item C in the new position immediately, before the server responds.

- Server returns 200 → UI stays as shown, other users receive WebSocket update
- Server returns 409 → UI rolls back to original order, toast: "Could not reorder — item was updated by another user"
- Server returns 500 → UI rolls back, no reorder event published

---

## Optimistic Locking — Same Item Concurrency

Each item has a `version` column.

Flow:

1. Client reads Item C at `version = 5`
2. Submits reorder with `version = 5`
3. Server executes:

```sql
UPDATE items
SET position = 1500, version = version + 1
WHERE id = 'item-c' AND version = 5
```

4. `rows_affected = 1` → success
5. `rows_affected = 0` → HTTP 409 Conflict → UI rolls back

This protects against: User 1 and User 2 both moving **Item C** simultaneously.

---

## Critical Gap — Different Items, Same Position

Item-level versioning does NOT protect the ordering structure itself.

**Scenario:**

```
Initial: A=1000, B=2000
User 1 moves C between A and B → calculates 1500
User 2 moves D between A and B → calculates 1500
Both update different rows → both succeed
Result: C=1500, D=1500 (corrupted order)
```

**Solutions:**

**Option 1 — Unique constraint + retry (preferred)**

```sql
UNIQUE(board_id, position)
```

One write succeeds, the other gets a constraint violation. The failing request recalculates position based on the latest ordering and retries.

**Option 2 — Group-level ordering version**

Each group has its own `ordering_version`. All reorders within a group include the group version in the conditional update. If two reorders hit the same group simultaneously, one wins and the other retries.

**Option 3 — Scope-level lock (advisory lock)**

Use PostgreSQL advisory locks at the board or group level to serialise reorders within that scope. Lower concurrency but simpler.

**Key interview point:** Optimistic locking on Item C solves `C vs C`. It does not solve `C vs D` when both move to the same location. The shared ordering structure is its own concurrency boundary.

---

## High-Level Architecture

```
User (drag + drop Item C)
        ↓
    Frontend (optimistic UI update)
        ↓
  POST /boards/{boardId}/items/{itemId}/reorder
  { afterItemId: "A", beforeItemId: "B", version: 5 }
        ↓
   Load Balancer / API Gateway
        ↓
    Board Service
        ↓
   PostgreSQL Write Primary
   Transaction:
     1. Validate neighbors (A and B positions)
     2. Calculate new rank
     3. UPDATE items SET position=1500, version=6 WHERE id='C' AND version=5
     4. INSERT into outbox (event: ItemReordered)
        ↓
   ┌──────────────────────────┐
   │                          │
Redis Pub/Sub           Outbox Publisher
(real-time path)        (async path)
   │                          │
WebSocket Servers           Kafka
   │                    ┌────┴────┐
All connected users   Automation  Audit  Analytics
on this board
```

---

## Transactional Outbox — Reliable Kafka Publishing

Dual-write problem: DB write succeeds, service crashes before publishing to Kafka. Event is lost.

Solution: write the event to an outbox table in the **same transaction** as the position update.

```sql
BEGIN TRANSACTION
  UPDATE items SET position=1500, version=6 WHERE id='C' AND version=5
  INSERT INTO outbox (event_type, payload) VALUES ('ItemReordered', {...})
COMMIT
```

A separate Outbox Publisher process reads unpublished outbox events and publishes to Kafka. If Kafka is unavailable, events stay in the outbox and are retried. If the publisher crashes, the events remain persisted and another instance picks them up.

This guarantees at-least-once delivery to Kafka without dual-write risk.

---

## Redis Pub/Sub — Real-Time Update

After successful DB write, the API server publishes to a board-specific Redis channel:

Channel: `board:{boardId}`

Message:

```json
{
  "type": "ITEM_REORDERED",
  "itemId": "C",
  "afterItemId": "A",
  "beforeItemId": "B",
  "newPosition": 1500,
  "version": 6,
  "movedBy": "user-1"
}
```

WebSocket servers subscribe to `board:{boardId}` and push to all connected clients.

**Redis Pub/Sub is not durable.** If a client is disconnected when the event is published, it misses the update.

**Reconciliation on reconnect:** each client tracks `lastSeenBoardVersion`. On reconnect, it compares with the current server version. If stale, it fetches the latest ordering.

Target real-time propagation: sub-100ms end-to-end (Redis fan-out + WebSocket delivery over network).

---

## Read Replica — Consistency Caveat

Writes go to the primary. Board reads go to the read replica.

Replication lag means a user who just reordered an item may see the old position if they immediately read from the replica.

Mitigations:
- Route a user's own read immediately after their write to the primary
- Use WebSocket update as the source of truth for the UI (not a board refetch)
- Acknowledge replication lag in the interview and explain the chosen trade-off

---

## Database — PostgreSQL

```sql
items (
  id          UUID PRIMARY KEY,
  board_id    UUID NOT NULL,
  position    DECIMAL NOT NULL,
  version     INT NOT NULL DEFAULT 0,
  updated_at  TIMESTAMPTZ,
  updated_by  UUID,
  UNIQUE(board_id, position)
)

CREATE INDEX idx_items_board_position ON items (board_id, position);

outbox (
  id          UUID PRIMARY KEY,
  event_type  VARCHAR NOT NULL,
  payload     JSONB NOT NULL,
  published   BOOLEAN DEFAULT FALSE,
  created_at  TIMESTAMPTZ DEFAULT NOW()
)
```

---

## Failure Handling

| Failure | Behaviour |
|---|---|
| DB write fails | API returns 500, UI rolls back, no event published |
| 409 Conflict (same item) | UI rolls back, toast shown, user retries manually |
| Unique constraint violation (same position) | Retry with recalculated position |
| Redis Pub/Sub fails | Reorder committed, other users miss real-time update, reconcile on reconnect |
| WebSocket server crashes | Client reconnects, compares version, refetches if stale |
| Kafka unavailable | Outbox retains event, publisher retries later, reorder already committed |
| Outbox publisher crashes | Events stay in PostgreSQL, another instance continues |

---

## Observability

| Signal | What to monitor |
|---|---|
| 409 conflict rate | High = heavy concurrent reordering on same items |
| Unique constraint violation rate | High = concurrent moves to same position, check uniqueness strategy |
| Rebalancing frequency | High = gap strategy needs tuning or initial gaps too small |
| WebSocket delivery latency | p50, p95, p99 — target p99 < 100ms |
| Kafka consumer lag | Automation / audit / analytics falling behind |
| DB write latency | Spikes = lock contention or rebalancing activity |
| Redis Pub/Sub error rate | Real-time propagation issues |
| Outbox queue depth | Unpublished events accumulating = Kafka publishing issue |

---

## Trade-offs

| Decision | Why | Alternative |
|---|---|---|
| DECIMAL position + local rebalance | O(1) reorder, rare and scoped rebalancing | Sequential integers — O(n) writes per move |
| Client sends intent (afterItemId/beforeItemId) | Server owns position calculation, prevents stale client errors | Client sends position — risk of stale state |
| Optimistic locking (version) | No upfront blocking, high concurrency | Pessimistic locking — serialises writes, lower throughput |
| Unique constraint + retry for same-position conflict | Simple, database enforces ordering integrity | Group-level version — more complex but handles broader concurrency |
| Transactional Outbox | No dual-write risk, guaranteed Kafka delivery | Direct Kafka publish — event lost if service crashes mid-flight |
| Redis Pub/Sub for real-time | Sub-millisecond distribution to WebSocket servers | Kafka — durable but not suited for real-time UI push |
| Optimistic UI | Instant perceived performance | Wait for server — visible network latency |
| PostgreSQL | ACID, conditional updates, ordered queries | DynamoDB — harder to do version-conditional updates cleanly |

---

## Diagram

See `item-reordering.excalidraw` — draw manually in Excalidraw.

Components to include:
- User drag + drop
- Frontend (optimistic UI + rollback path)
- Load Balancer / API Gateway
- Board Service
- PostgreSQL Write Primary (with outbox table)
- Outbox Publisher
- PostgreSQL Read Replica
- Redis Pub/Sub (channel: board:{boardId})
- WebSocket Servers
- Connected Users (User 2, User 3)
- Kafka
- Automation Engine, Audit Trail, Analytics
- Conflict path: 409 → UI rollback → toast
- Reconnect path: version check → refetch if stale
