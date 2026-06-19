# Notification System Design — Learnings

## What This Design Teaches

The notification system is one of the most important designs to master because it covers:
- Event-driven architecture
- Async messaging at scale
- Idempotency and deduplication
- Retry and circuit breaker patterns
- Outbox Pattern for guaranteed delivery
- Multi-channel delivery (email, SMS, push)

**You built this at Stepstone — 10 million notifications per day across 16 European markets. This is your strongest story.**

---

## Core Problem

A user action (job application, booking confirmation, password reset) must trigger a notification reliably. The notification must be delivered exactly once even if the system crashes, retries, or has network failures.

---

## Architecture Overview

```
Trigger (HTTP event / Kafka / S3 / Hangfire batch)
 ↓
API / Event Consumer
 ↓
Outbox Table (PostgreSQL) — atomic write
 ↓
Outbox Worker — polls pending events
 ↓
SQS
 ↓
Notification Consumer — circuit breaker, retry
 ↓
SES / SNS / Push provider
 ↓
User (email / SMS / push)
```

---

## Four Trigger Patterns

| Pattern | Trigger | Use Case |
|---|---|---|
| HTTP | REST API call | Password reset, user action |
| Kafka | Event stream | Real-time job match, booking confirmed |
| S3 | File upload | Bulk import, CSV processing |
| Hangfire batch | Scheduled job | Daily digest, reminders |

---

## Outbox Pattern — Guaranteed Delivery

**Problem:** Service writes to DB but crashes before publishing to SQS. Message lost.

**Solution:**
```
In ONE database transaction:
1. Write notification intent to outbox table (status = pending)
2. Both succeed or both rollback — atomic

Separately:
3. Outbox Worker polls outbox table every few seconds
4. Publishes pending events to SQS
5. Marks as published = true
```

If service crashes between steps 1 and 4, the outbox row survives. Worker picks it up on restart.

**This guarantees at-least-once delivery.**

---

## Idempotency — Prevent Duplicate Notifications

At-least-once delivery means the same message could be processed twice. You must deduplicate.

**Solution — Redis idempotency key:**
```
Key: notification:{userId}:{eventType}:{eventId}
TTL: 24 hours

Before processing:
→ Check Redis for key
→ Key exists → already processed → skip
→ Key missing → process → set key in Redis
```

Same message delivered twice — second delivery hits Redis key, skipped. User gets exactly one notification.

---

## Circuit Breaker — Polly

If SES or downstream provider is down, do not keep hammering it. Apply circuit breaker.

```
States:
Closed   → requests flow normally
Open     → requests fail immediately (fast fail)
Half-Open → allow one test request through

Closed → too many failures → Open
Open → wait timeout → Half-Open
Half-Open → success → Closed
Half-Open → failure → Open
```

**Why it matters:** Without circuit breaker, a slow SES causes your service threads to pile up waiting. Connection pool exhausts. Service goes down. Circuit breaker fails fast and protects your service.

---

## Retry with Exponential Backoff

Transient failures (network blip, SES rate limit) should be retried. But do not retry immediately.

```
Attempt 1: fail → wait 1 second
Attempt 2: fail → wait 2 seconds
Attempt 3: fail → wait 4 seconds
Attempt 4: fail → move to DLQ
```

Exponential backoff prevents hammering a struggling downstream service.

---

## Dead Letter Queue (DLQ)

Messages that fail after all retries move to DLQ. They are not lost — they wait for investigation.

```
SQS → Consumer fails 3 times → message moves to DLQ
```

Ops team investigates DLQ. Fix the issue. Replay messages.

---

## Multi-Channel Delivery

| Channel | Provider | Use Case |
|---|---|---|
| Email | AWS SES | Booking confirmations, receipts |
| SMS | SNS / Twilio | OTP, urgent alerts |
| Push | Firebase FCM | Mobile app notifications |

Route by notification type and user preference. User may opt out of email but keep push.

---

## Scale Considerations

**10 million notifications per day = ~116 per second average. Peak could be 10x.**

- SQS scales automatically — no capacity to manage
- Multiple Consumer instances in ECS — scale by CPU or queue depth
- SES sending limits — request quota increase, use dedicated IPs for high volume
- Partition Kafka by userId — ensures ordering per user, avoids hot partitions

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Sync vs Async | Async via SQS | Decouple trigger from delivery, handle failures independently |
| At-least-once vs exactly-once | At-least-once + idempotency | Exactly-once is hard, cheaper to deduplicate at consumer |
| Push vs pull | SQS pull | Consumer controls pace, no overload |
| Retry strategy | Exponential backoff | Avoids hammering struggling downstream |
| Failure handling | DLQ | No data loss, investigate and replay |

---

## Topics Covered in This Design

| Topic | Concept |
|---|---|
| Event-driven architecture | Triggers, consumers, decoupling |
| Outbox Pattern | Atomic write, guaranteed delivery |
| Idempotency | Redis deduplication, exactly-once semantics |
| Circuit breaker | Polly, Closed/Open/Half-Open states |
| Retry | Exponential backoff, transient failures |
| Dead Letter Queue | Failure handling, no data loss |
| Multi-channel | SES, SNS, FCM routing |
| Scaling | SQS auto-scale, ECS horizontal scaling |

---

## Key Interview Phrases

- "The Outbox Pattern guarantees at-least-once delivery by writing the event atomically with the business data in the same transaction"
- "We use a Redis idempotency key to convert at-least-once into exactly-once at the consumer level"
- "Circuit breaker protects our service from cascading failures when SES is degraded — we fail fast instead of exhausting our thread pool"
- "DLQ ensures no message is ever permanently lost — failed messages wait for investigation and replay"
- "We partition Kafka by userId to maintain ordering per user and avoid hot partitions on popular event types"
