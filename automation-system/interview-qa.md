# Automation System — monday.com

---

## Clarifications

- Triggers are board events (status changed, item created, item moved) and time-based (due date arrives)
- Automation effect does not need to be instant — 2 to 5 seconds acceptable
- Idempotency required — same event must not fire the same automation twice
- Audit trail required — 6 months retention
- Notification delivery (email, push, in-app) is handled by a separate notification service — not in scope
- One event can match multiple automation rules on the same board
- Scope: 3 trigger types, 3 action types

---

## Back of Envelope

| Metric | Value |
|---|---|
| Average throughput | 10,000 events/sec |
| Peak throughput | 50,000 events/sec |
| Daily volume | 864M events/day |
| Per-event size | ~1KB |
| Daily storage | ~864GB/day |
| 6-month retention | ~157TB |
| With one read replica | ~315TB |

---

## High-Level Architecture

```
Board Service / User Action
        ↓
      Kafka
  (partitioned by BoardId)
        ↓
  Automation Engine
  (consumes Kafka, matches rules)
        ↓
       SNS
        ↓
  ┌─────┬──────────────┬────────────────┐
  SQS   SQS            SQS
  │     │              │
  Task  Notification   Audit Trail
  Performer  Service   Service
  │
  Board Service API
  (assign user / move item)
```

---

## Kafka — Partitioning

Partition key: **BoardId**

All events for a given board land on the same partition, preserving ordering within the board.

Hot partition risk: if a single board is extremely active, that partition becomes a bottleneck. Mitigation: use a composite key such as `boardId + itemId` for high-traffic boards.

Kafka provides at-least-once delivery. Idempotency layer handles deduplication.

---

## Rule Store — DynamoDB

Rules are stored in DynamoDB.

Key: `boardId`
Value: JSON array of automation rules configured for that board

Example rule document:
```json
{
  "boardId": "board-123",
  "rules": [
    {
      "ruleId": "rule-abc",
      "trigger": { "type": "STATUS_CHANGED", "toStatus": "Done" },
      "action": { "type": "NOTIFY", "userId": "user-456" }
    }
  ]
}
```

**Redis cache sits in front of DynamoDB.** The Automation Engine checks Redis first. On cache miss, it fetches from DynamoDB and populates the cache.

Cache invalidation: when a user creates, updates, or deletes an automation rule, the cache for that boardId is invalidated or updated immediately.

---

## Idempotency — Redis

Before processing an event, the Automation Engine checks Redis using the Kafka message ID as the idempotency key.

- Key exists → already processed → skip
- Key does not exist → process and write key to Redis with TTL (24 hours)

Prevents duplicate automation execution on retry.

---

## SNS Fan-out → SQS per Service

After the Automation Engine matches rules and determines required actions, it publishes to SNS.

SNS fans out to a dedicated SQS queue per downstream service:

- Task Performer SQS — handles assign user, move item to Overdue
- Notification SQS — handles notify manager/user (delivers to notification service)
- Audit Trail SQS — handles writing execution history

Each service polls its own SQS queue independently. This gives each service independent retry control, DLQ, and consumption rate.

---

## Failure Handling

SQS visibility timeout: **30 to 60 seconds** — must be longer than the maximum expected processing time per message.

If a consumer fails to process and does not delete the message, SQS makes the message visible again after the visibility timeout expires.

After max retries (e.g. 3), the message moves to a **Dead Letter Queue (DLQ)**.

A CloudWatch alarm fires on DLQ depth > 0. On-call engineer investigates root cause, fixes the issue, and redrives messages from DLQ back to the main queue.

---

## Time-Based Trigger — Scheduler

"WHEN a due date arrives" is not triggered by a user action. It requires a dedicated scheduler.

**Approach:**

A dedicated `scheduled_events` table stores items with a due date:

```
scheduled_events
  item_id       (PK)
  board_id
  due_date      (indexed)
```

An index on `due_date` allows efficient range queries.

The scheduler runs on a short interval (e.g. every minute) and queries:

```sql
SELECT * FROM scheduled_events
WHERE due_date BETWEEN now AND now + INTERVAL '1 minute'
```

Matching items are published as events to Kafka, entering the same pipeline as board events.

The scheduler reads from the **read replica** — never from the write database.

---

## Audit Trail

The Audit Trail Service subscribes to SNS via its own SQS queue and writes an execution record for every automation that fires.

Record includes: boardId, ruleId, triggering event, action taken, timestamp, result (success/failure).

Retention: 6 months. After 6 months, records are deleted or archived to cold storage (e.g. S3).

**The audit trail is the first diagnostic tool when an automation is reported as not firing.** Check whether a record exists. If yes, problem is downstream. If no, problem is upstream.

---

## Observability

| Signal | What to monitor |
|---|---|
| Kafka consumer lag | Automation Engine falling behind |
| SQS queue depth | Downstream services backing up |
| DLQ depth | Failed messages requiring investigation |
| Redis hit/miss ratio | Cache effectiveness |
| Automation execution rate | Drop may indicate upstream event failure |

Logs: structured logs with boardId, ruleId, eventId, and trace ID on every event.

Traces: distributed tracing across Kafka → Automation Engine → SNS → SQS → downstream services.

---

## Trade-offs

| Decision | Why | Alternative |
|---|---|---|
| Kafka over SQS for ingestion | High throughput, replay, partitioning by BoardId | SQS — simpler but no replay, weaker ordering |
| DynamoDB for rule store | O(1) lookup by boardId, scales well | PostgreSQL — relational but slower for this key-value pattern |
| Redis cache for rules | Avoid DynamoDB read on every event at 10K RPS | No cache — DynamoDB can handle it but at higher cost and latency |
| SNS fan-out to SQS per service | Independent retry, DLQ, consumption rate per service | Single shared queue — simpler but no isolation |
| Dedicated scheduler for due dates | Efficient indexed query, decoupled from board events | DynamoDB TTL streams — elegant but less control over timing precision |

---

## Diagram

See `automation-system.excalidraw` — draw manually in Excalidraw.

Components to include:
- Board Service / User Action
- Kafka (partitioned by BoardId)
- Automation Engine (with Redis cache + DynamoDB rule store)
- Redis (idempotency + rule cache)
- DynamoDB (rule store)
- SNS
- SQS per service (Task Performer, Notification, Audit Trail)
- Task Performer → Board Service API
- Notification Service
- Audit Trail DB
- DLQ
- Scheduler (with scheduled_events table + read replica)
- CloudWatch / Alerting
