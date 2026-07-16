# Automation System — monday.com

---

## Clarifications

- Triggers: board events (status changed, item created, item moved) and time-based (due date arrives)
- 2 to 5 seconds execution latency acceptable
- One event can match multiple automation rules
- Idempotency required — same event must not execute the same automation rule twice
- Audit trail required — 6 months retention
- Notification delivery (email, push, in-app) handled by a separate Notification Service
- Scope: 3 trigger types, 3 action types

---

## Back of Envelope

| Metric | Value |
|---|---|
| Average throughput | 10,000 events/sec |
| Peak throughput | 50,000 events/sec |
| Daily volume | 864M events/day |
| Per-event size | ~1KB |
| Daily raw event storage | ~864GB/day |
| 6-month raw event storage | ~157TB |

Note: this is raw incoming event volume. Only events matching automation rules generate execution records. For accurate audit storage, ask for the automation match rate and average matched rules per event.

---

## High-Level Architecture

```
Users
  ↓
Board Service
  ↓
Kafka: BoardEvents (key = BoardId)
  ↓
Automation Engine [AE1 ... AEN]
  ├── Redis Rule Cache
  └── DynamoDB Rule Store
  ↓
Durable Execution Record (EventId + RuleId)
  ↓
Transactional Outbox
  ↓
Outbox Publisher
  ↓
SNS
  ├── Task SQS → Task Worker → Board API
  ├── Notification SQS → Notification Worker → Notification Service
  └── Audit SQS → Audit Worker → Execution Store

Scheduler → Kafka (time-based triggers, same pipeline)

CloudWatch / Datadog (Kafka lag, SQS depth, DLQ depth, latency, error rate)
```

---

## Kafka — Partitioning

**Default partition key: BoardId**

Events for the same board go to the same partition, preserving board-level ordering.

**Hot partition risk:** an extremely active board overloads one partition.

**Alternative: BoardId + ItemId**

Better distribution and parallelism. Loses board-level ordering, keeps item-level ordering.

**Interview answer:**
"The partition key should reflect the required ordering boundary. If ordering is required across the full board I use BoardId. If ordering is only required per item I use BoardId + ItemId for greater parallelism, accepting that board-level ordering is lost."

**Scaling limit:** maximum useful consumer parallelism in one consumer group equals the partition count. 10 partitions = at most 10 actively consuming consumers. Scale partitions when adding consumers.

---

## Rule Store — DynamoDB + Redis Cache

DynamoDB is the durable rule store.

Access pattern — prefer BoardId + TriggerType as the key:

```
PK: BOARD#123
SK: TRIGGER#STATUS_CHANGED#RULE#456
```

Redis caches rules per board per trigger type:

```
Cache key: rules:{BoardId}:{TriggerType}
Example:   rules:123:STATUS_CHANGED
```

Flow:

```
Automation Engine
  ↓
Redis (cache hit → evaluate rules)
  ↓ (cache miss)
DynamoDB → populate Redis → evaluate rules
```

On rule create, update, or delete: invalidate or update the relevant cache entry immediately.

**Important:** Redis is a performance layer, not a correctness layer. Do not rely on Redis alone for durable idempotency. Redis can be evicted or restarted.

---

## Idempotency — EventId + RuleId

One event can match multiple rules. Using EventId alone as the idempotency key is wrong.

**Correct idempotency boundary: EventId + RuleId**

```
Event E1 matches R1, R2, R3

Idempotency keys:
  E1:R1
  E1:R2
  E1:R3
```

Redis provides fast idempotency check:

```
Key: automation:idempotency:{eventId}:{ruleId}
TTL: 24 hours
```

But Redis should not be the only mechanism — use durable execution records for correctness.

---

## Durable Execution Records

Create a durable execution record for every matched rule before publishing any action.

```sql
automation_executions (
  execution_id   UUID PRIMARY KEY,
  event_id       UUID NOT NULL,
  rule_id        UUID NOT NULL,
  board_id       UUID NOT NULL,
  action_type    VARCHAR NOT NULL,
  status         VARCHAR NOT NULL,  -- MATCHED, CREATED, DISPATCHED, PROCESSING, SUCCEEDED, FAILED
  failure_reason TEXT,
  created_at     TIMESTAMPTZ,
  updated_at     TIMESTAMPTZ,
  UNIQUE(event_id, rule_id)
)
```

If Kafka redelivers an event, the UNIQUE constraint prevents creating duplicate executions. The existing record is detected and the duplicate is skipped.

**Audit trail:** this table IS the audit trail. Track lifecycle:

```
MATCHED → CREATED → DISPATCHED → PROCESSING → SUCCEEDED / FAILED
```

When a customer says an automation did not fire:
- No execution record → investigate upstream (event generation, Kafka, rule matching)
- Execution exists, incomplete → investigate downstream (worker, action execution)
- Status FAILED → inspect failure reason and DLQ

---

## Kafka Offset Commit Strategy

Precise statement:

"The consumer pipeline is configured for at-least-once processing. The Automation Engine commits the Kafka offset only after successfully creating durable execution records."

Failure scenario:

```
Consume E1
Match R1, R2
Create E1:R1, E1:R2
★ Crash before offset commit
Kafka redelivers E1
E1:R1 and E1:R2 already exist → skip duplicates
```

This guarantees at-least-once processing with durable idempotency preventing duplicate execution.

---

## Transactional Outbox

Dual-write problem: Automation Engine writes execution record to DB, then crashes before publishing to SNS. Action is lost.

Solution: write execution record and outbox event atomically in one transaction.

```sql
BEGIN TRANSACTION
  INSERT INTO automation_executions (execution_id, event_id, rule_id, status, ...)
  INSERT INTO outbox (event_type, payload)
COMMIT
```

Outbox Publisher reads unpublished outbox events and publishes to SNS. If SNS is unavailable, events remain in the outbox and are retried. If the publisher crashes, another instance continues.

**Interview strategy:** do not draw the Outbox immediately. Introduce it when asked "what happens if the Automation Engine crashes between saving state and publishing the action?"

---

## SNS Fan-out → SQS per Service

SNS fans out to a dedicated SQS queue per downstream service:

- Task SQS → Task Worker → Board API (assign user, move item)
- Notification SQS → Notification Worker → Notification Service
- Audit SQS → Audit Worker → Execution Store

Benefits: independent scaling, independent retry, independent DLQ, failure isolation. Notification slowness does not block Task or Audit processing.

**Visibility timeout:** set comfortably longer than expected processing time. If processing takes 10 seconds, set to 30 to 60 seconds.

**Retry → DLQ:** after max receive attempts (e.g. 3), message moves to DLQ. CloudWatch alarm on DLQ depth > 0. On-call investigates, fixes, redrives.

---

## Downstream Worker Idempotency

Workers must also be idempotent. SQS delivers at-least-once.

Example: Task Worker moves item to Overdue, then crashes before deleting the SQS message. SQS redelivers.

Use ExecutionId or ActionId as the downstream idempotency key. If already executed: acknowledge and skip. Otherwise: execute the action.

---

## Time-Based Trigger — Scheduler

"WHEN a due date arrives" is not triggered by a user action. Requires a dedicated Scheduler.

```sql
scheduled_events (
  scheduled_event_id   UUID PRIMARY KEY,
  item_id              UUID NOT NULL,
  board_id             UUID NOT NULL,
  due_date             TIMESTAMPTZ NOT NULL,   -- indexed
  status               VARCHAR NOT NULL
)

CREATE INDEX idx_due_date ON scheduled_events (due_date) WHERE status = 'PENDING';
```

Scheduler periodically queries due events and publishes DueDateReached events to Kafka. Same Automation Engine pipeline handles them identically to board events.

**Read replica and replication lag:** using a read replica for scheduler queries is good for scale, but replication lag can make newly created due dates temporarily invisible.

Mitigation: use overlapping query windows.

Example: scheduler at 10:01 queries due dates from 09:59 to 10:02. Duplicates are acceptable because EventId + RuleId idempotency prevents duplicate execution.

**Interview answer:** "I use a read replica to offload scheduler queries where slight replication lag is acceptable. I use overlapping query windows to prevent missed events, and idempotent processing to handle duplicates safely."

---

## Failure Scenarios

| Failure | Behaviour |
|---|---|
| Kafka redelivers same event | EventId + RuleId unique constraint prevents duplicate execution |
| Automation Engine crashes before offset commit | Kafka redelivers; existing execution record skips duplicate |
| Crash between DB write and SNS publish | Transactional Outbox retains event; publisher retries |
| Redis is down | Fall back to DynamoDB for rule lookup; do not rely on Redis for durable idempotency |
| Task Worker executes action, crashes before SQS delete | SQS redelivers; ExecutionId idempotency prevents duplicate side effect |
| Notification Service is down | Notification SQS backs up; Task and Audit continue independently |
| Scheduled event selected twice | ScheduledEventId + RuleId idempotency prevents duplicate execution |
| Read replica replication lag | Overlapping scheduler windows + idempotency covers the gap |
| Hot board partition | Consider BoardId + ItemId partition key, accepting loss of board-level ordering |
| Kafka consumer lag increases | Scale consumers up to partition count; scale partitions if needed |

---

## Observability

| Signal | What to monitor |
|---|---|
| Kafka consumer lag | Automation Engine backlog, autoscaling signal |
| SQS queue depth | Downstream backlog |
| DLQ depth > 0 | Failed messages requiring investigation |
| Redis hit/miss ratio | Cache effectiveness |
| Automation execution rate | Unexpected drop = upstream event failure |
| Automation execution latency | Verify 2 to 5 second target |
| Error rate | Detect failures across pipeline |

Structured logs: every log entry includes BoardId, RuleId, EventId, ExecutionId, TraceId.

Distributed trace: Board Service → Kafka → Automation Engine → Outbox → SNS → SQS → Worker.

---

## Interview Presentation Strategy

**Do not start with every advanced component.**

Start with:

```
Board Service → Kafka → Automation Engine → SNS → SQS → Workers
```

Then add:
- DynamoDB rule store
- Redis cache
- Scheduler for time-based triggers

Then discuss:
- Partitioning trade-off (BoardId vs BoardId + ItemId)
- Scaling (consumer count vs partition count)
- Rule matching

When asked about duplicates:
→ Introduce EventId + RuleId idempotency

When asked about durable correctness:
→ Introduce durable Automation Execution records

When asked about crashing between DB write and publishing:
→ Introduce Transactional Outbox

This demonstrates the ability to evolve the architecture based on requirements rather than over-engineering upfront.

---

## Trade-offs

| Decision | Why | Alternative |
|---|---|---|
| Kafka over SQS for ingestion | High throughput, replay, partition-based ordering, consumer groups | SQS — simpler but no replay, weaker ordering |
| DynamoDB for rule store | Scalable, efficient known-key access patterns | PostgreSQL — relational but slower for this key-value pattern |
| Redis cache for rules | Low-latency, reduces DynamoDB reads at 10K RPS | No cache — DynamoDB can handle it but at higher cost and latency |
| EventId + RuleId idempotency | One event can match multiple rules; EventId alone is insufficient | EventId only — causes duplicate actions for multi-rule events |
| Durable execution records | Stronger than Redis idempotency alone; full audit trail | Redis only — eviction or restart loses idempotency |
| Transactional Outbox | Prevents lost SNS events on crash between DB write and publish | Direct publish — event lost if process crashes mid-flight |
| SNS fan-out to SQS per service | Independent retry, DLQ, consumption rate per service | Single shared queue — simpler but no failure isolation |
| Dedicated Scheduler for due dates | Efficient indexed query, same Kafka pipeline | DynamoDB TTL streams — elegant but less timing control |
| BoardId partition key | Board-level ordering | BoardId + ItemId — better distribution, loses board ordering |

---

## Diagram

See `automation-system.excalidraw`.

Components to include:
- Board Service / User Action
- Kafka (partitioned by BoardId)
- Automation Engine (with Redis cache + DynamoDB rule store)
- Redis (idempotency + rule cache)
- DynamoDB (rule store)
- Durable Execution Store (PostgreSQL)
- Transactional Outbox
- Outbox Publisher
- SNS
- SQS per service (Task, Notification, Audit)
- Task Worker → Board Service API
- Notification Worker → Notification Service
- Audit Worker → Execution Store
- DLQ for each queue
- Scheduler (with scheduled_events table + read replica)
- CloudWatch / Datadog
