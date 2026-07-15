# Notification System — Variant 2: Upstream Publishes Directly to SQS

> When to use: upstream services are trusted and can publish reliably to SQS directly.
> No Outbox Pattern needed. Simpler, fewer moving parts.
> See notes.md (Variant 1) for the Outbox Pattern approach.

---

## Clarifications to Ask Upfront

- Do events include audience details (email, phone, channel preference)? Or call a user service?
- Instant vs batch delivery — security alerts instant, marketing acceptable delay?
- Expected latency per priority tier?
- Can duplicates arrive from upstream? Who handles idempotency?
- Are events pushed (HTTP) or pulled (queue)? — Always push to a queue at this scale.

---

## Back of Envelope (10,000 events/second, 10M users)

- 864M events/day
- 86 events per user per day
- ~1KB per event = 864GB/day, ~315TB/year, ~946TB with 3x replication
- Throughput: ~10MB/second

---

## High Level Design

```
Upstream Services
  → SQS Priority Queue   (security alerts — instant, < 1s)
  → SQS Standard Queue   (marketing — batch, < 1 hour)
        |
  ECS Consumer
  → Redis idempotency check (key = SQS MessageId, TTL = 24h)
  → SNS Topic (fan-out)
        |
  ┌──────────────┬──────────────┐
Email Service  SMS Service  Push Service
(AWS SES)    (Twilio/SNS)  (FCM / APN)
        |
    Recipients
        |
CloudWatch metrics → SNS Alert (ops)
DLQ (failed after retries)
Delivery Status DB (PostgreSQL)
```

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Ingestion | SQS (upstream publishes directly) | Decouples producer/consumer, handles bursts, no polling |
| Priority | Two separate SQS queues | Security alerts never stuck behind marketing bulk |
| Fan-out | SNS to Email/SMS/Push | Each channel scales independently |
| Idempotency | Redis (SQS MessageId as key, TTL 24h) | Fast dedup, at-least-once → effectively-once |
| Exactly-once fallback | PostgreSQL as source of truth behind Redis | If Redis crashes, DB guarantees no duplicates — adds complexity |
| Failures | DLQ after retry exhaustion | No data loss, investigate and replay |
| Observability | CloudWatch + SNS Alert | Ops paged on queue depth, delivery rate drop, error spikes |

---

## Idempotency Flow

```
Consumer receives message from SQS
→ Check Redis: exists(MessageId)?
  → Yes → skip, already processed
  → No  → publish to SNS → write MessageId to Redis (TTL 24h)
```

**Edge case:** Consumer publishes to SNS then crashes before writing Redis key. On retry it processes again — duplicate sent.

**Fix for exactly-once:** Write idempotency key to PostgreSQL first (durable), use Redis as fast cache in front.

**Trade-off:** At-least-once + Redis = simpler, rare duplicates acceptable. Exactly-once = complex, required when business says no duplicates ever.

---

## Priority Queue Pattern

Two SQS queues, two consumer groups:

- Priority consumer: dedicated ECS tasks, high concurrency, processes instantly
- Standard consumer: fewer tasks, can batch or throttle, processes within 1 hour

Never mix priorities in one queue — a marketing blast would delay security alerts.

---

## Trade-offs to State in Interview

| Topic | Trade-off |
|---|---|
| At-least-once vs exactly-once | Simplicity vs guarantee — let business decide |
| Redis vs PostgreSQL for idempotency | Speed vs durability |
| SNS fan-out vs per-channel queues | Simpler routing vs independent scaling per channel |
| Two SQS queues vs one | Priority isolation vs operational complexity |

---

## Senior Behaviour in System Design Interview

- Make the call yourself, then justify — do not ask the interviewer to decide
- Only ask about constraints you cannot assume: scale, latency, existing infra, compliance
- State trade-offs explicitly — interviewers want to hear you name them
- The design evolves through discussion — no single correct answer
- Propose, justify, check alignment: "I would use SQS here because it decouples the producer from the consumer and handles burst traffic. Does that fit your infrastructure?"
