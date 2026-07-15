# Webhook Delivery System — Mock Interview Q&A

## Problem Statement

Design a system that reliably delivers events from internal platforms to external services via HTTP webhooks. When an event occurs (e.g. payment completed, account updated), the system must send an HTTP POST to a URL registered by the downstream service. Delivery must be reliable — if the target is unavailable, retry. If retries are exhausted, alert the team.

---

## Clarifying Questions

**You:** How many events per second are we expecting?
**Interviewer:** 10,000 events per second at peak.

**You:** What is the delivery promise — at-least-once, exactly-once?
**Interviewer:** At-least-once. Idempotency is the downstream system's responsibility. Include an event_id in every payload.

**You:** Is this synchronous — does the upstream wait for delivery confirmation?
**Interviewer:** No — fire and forget. Return 202 Accepted immediately.

**You:** Do we need to transform or validate the event payload?
**Interviewer:** No — pass through as-is. Trust the upstream.

**You:** Retry policy — how many retries before we give up?
**Interviewer:** Exponential backoff, up to 5 retries. After 5 failures mark as DEAD and alert.

---

## Back of Envelope

- Peak: 10,000 events/sec
- Average payload: 1KB
- Per day: 10,000 × 1KB × 86,400 = 864GB/day
- Per year: ~315TB

**Decision:** Retain webhook logs for 30 days (compliance), then delete or archive to S3.
- Hot storage: 30 × 864GB = ~26TB — manageable with PostgreSQL

---

## Architecture

### Components

1. **Upstream Services** — Payment Service, Ledger, Account Balance — publish events to SQS
2. **SQS** — one queue, single source of truth for incoming events. DLQ for messages that fail ingestion.
3. **Ingestion Service** — polls SQS, looks up registered URL from DynamoDB, writes record to PostgreSQL with status PENDING
4. **Dispatcher** — polls PostgreSQL for PENDING records, makes HTTP POST to registered URL, updates status to DELIVERED or FAILED, retries with exponential backoff
5. **DynamoDB** — webhook registry: event_type → registered URL (key-value lookup)
6. **PostgreSQL** — webhook_deliveries table: event_id, status, retry_count, registered_url, payload, created_at
7. **CloudWatch Alarm / PagerDuty** — triggered by Dispatcher after 5 failed retries (DEAD status)

### Why split Ingestion and Dispatcher?

Ingestion must be fast — it just writes PENDING and moves on. If the Dispatcher were handling retries inline, a slow downstream endpoint would block new events. Separation keeps ingestion throughput high and retry logic isolated.

### Why DynamoDB for the registry?

The webhook registry lookup is a pure key-value operation: give me the URL for this event_type. DynamoDB is optimal — single-key lookup, fast, no joins needed.

### Why PostgreSQL for webhook_deliveries?

The Dispatcher queries by status: `SELECT * FROM webhook_deliveries WHERE status = 'PENDING' LIMIT 100`. This requires a query on a non-primary-key column — DynamoDB would need an expensive GSI scan. PostgreSQL with an index on status is the right tool.

---

## Status Flow

```
PENDING → DELIVERED (HTTP 2xx received)
PENDING → FAILED    (HTTP non-2xx or timeout, retry scheduled)
FAILED  → PENDING   (retry attempt)
FAILED  → DEAD      (after 5 retries — alert triggered)
```

---

## Deep Dive Questions

**Q: How do you prevent the same event being delivered twice?**
Every payload includes an event_id. Downstream systems can use this to deduplicate if needed. We guarantee at-least-once — exactly-once is their responsibility.

**Q: What if two Dispatcher instances pick up the same PENDING record?**
Use a database-level lock: `SELECT FOR UPDATE SKIP LOCKED`. This ensures only one Dispatcher instance processes a given record at a time. PostgreSQL supports this natively.

**Q: How does exponential backoff work?**
Retry 1: wait 10s, Retry 2: wait 30s, Retry 3: wait 1m, Retry 4: wait 5m, Retry 5: wait 30m. After retry 5 fails, mark DEAD and alert. This avoids hammering an already struggling downstream service.

**Q: What if the Ingestion Service cannot find a registered URL for an event_type?**
Drop the event and log a warning. No URL registered means no one is listening for that event type. Do not write to webhook_deliveries.

**Q: How does a downstream service register a webhook URL?**
Via a registration API — `POST /webhooks` with body `{ event_type, url }`. This writes to the DynamoDB registry. Out of scope for this design but worth mentioning.

**Q: What is the retention policy?**
Keep webhook_deliveries records for 30 days, then delete or archive to S3. This covers compliance requirements and keeps hot storage at ~26TB.
