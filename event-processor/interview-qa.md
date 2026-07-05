# Event Processor — Mock Interview Q&A

## Clarification Phase

**Interviewer:** Design a system that processes financial events where no event is ever lost and no event is processed twice.

**Praveen:** Before I start — a few clarifying questions.
- From where are the events flowing into this system? Is there an upstream service or a message queue?
- When you say no event lost — does that mean at-least-once delivery guarantee?
- Idempotency needs to be enforced so events are not processed twice — is that correct?
- How many events are we expecting per second?
- What type of events are these — payment events, transactions, ledger postings?

**Interviewer:** Events come from upstream services via a REST API — clients POST directly to your system. They are financial transaction events: debits, credits, transfers. Peak load is 5,000 events per second. At-least-once is correct. Every event has a unique `event_id` provided by the client.

**Praveen:** Two more questions — what is the expected processing latency? And what downstream systems need to consume these events?

**Interviewer:** Processing must complete within 5 seconds of receipt. Downstream consumers are: Ledger, Audit, and Reconciliation.

---

## Requirements

**Praveen:** Let me state the requirements before I go to back of envelope.

Functional:
- Accept financial events via POST /events
- Deliver every event to Ledger, Audit, and Reconciliation
- At-least-once delivery — no event lost
- Idempotent processing — no event processed twice

Non-functional:
- CP — consistency over availability, this is financial data
- Zero data loss — events must survive crashes
- 5 second end-to-end SLA
- Full audit trail — every event queryable

---

## Back of Envelope

**Praveen:**
- Average TPS: 500, Peak TPS: 5,000
- 500 * 86,400 = 43.2M events per day
- 1 event = ~1KB
- 43.2M * 1KB = 43.2GB per day
- 43.2GB * 365 = ~15.7TB per year
- Peak write throughput: 5,000 * 1KB = 5MB/sec — manageable, no sharding needed at this scale

---

## High Level Design

**Praveen:** I do not see a reason to store events in a relational database long term. The core of this system is accepting the event, guaranteeing it is not lost, and pushing it to downstream consumers reliably.

My approach:
- Client calls POST /events via API Gateway to Event API Service
- Event API Service writes the event to an outbox table in the database with status PENDING. The outbox table has a UNIQUE constraint on `event_id` to prevent duplicates.
- Return 202 Accepted immediately with the `event_id` — the caller does not wait for processing.
- An Outbox Worker polls the outbox table for PENDING records and publishes them to SNS.
- SNS delivers to Ledger, Audit, and Reconciliation — each subscribes independently.

**Interviewer:** You said idempotency is checked at the outbox table level. How exactly does that work? What prevents two simultaneous requests with the same `event_id` both passing the check?

**Praveen:** A UNIQUE constraint on `event_id` in the outbox table. If a duplicate is inserted, the database throws a constraint violation. I catch it and return the original accepted response. The database enforces this atomically — there is no race condition.

**Interviewer:** SNS delivers at-least-once. A downstream consumer could receive the same event twice. How does each consumer protect itself?

**Praveen:** Each consumer maintains its own idempotency store. Check Redis cache first using `event_id` as the key with a TTL. If not found, fall back to the database as the source of truth. If already processed, skip. If not seen, process and write to both DB and Redis.

**Interviewer:** What HTTP status code and response body do you return to the caller?

**Praveen:** 202 Accepted. Response body:
```json
{ "event_id": "evt_123", "status": "ACCEPTED" }
```
The event will be processed under the hood. The caller can poll or use a webhook if they need confirmation.

---

## Failure Handling

**Interviewer:** What happens when the outbox worker crashes after publishing to SNS but before marking the record as PROCESSED?

**Praveen:** The worker restarts, sees the record still PENDING, and publishes again. SNS delivers twice. The consumers handle it via idempotency — but to minimise unnecessary duplicate publishes, the outbox record has three status transitions:

```
PENDING → PROCESSING → PROCESSED
```

The worker marks a record PROCESSING before it publishes. If it crashes mid-publish, on restart it sees PROCESSING records. It can safely republish because consumers are idempotent. A timeout — say 30 seconds in PROCESSING — triggers a retry. This prevents duplicate publishes in the common case while still guaranteeing delivery.

---

## Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Idempotency at ingestion | UNIQUE constraint on `event_id` | Database enforces atomically, no race condition |
| Delivery guarantee | Outbox pattern + SNS | Survives crashes, at-least-once to all consumers |
| API response | 202 Accepted | Decouples ingestion from processing, fast response |
| Consumer idempotency | Redis cache + DB fallback | Fast path avoids DB on every event |
| Outbox status | PENDING → PROCESSING → PROCESSED | Minimises duplicate publishes on worker restart |
| Consistency | CP | Financial data — correctness over availability |
