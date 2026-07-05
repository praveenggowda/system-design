# Banking System Design — Concepts and Q&A

---

## Topic 1: Double-Entry Bookkeeping

### What is it?
Every transaction touches two accounts — debit one, credit another. The books always balance.
For any transaction: SUM(DEBIT entries) = SUM(CREDIT entries). If they do not balance, reject the transaction.

### Database Structure

| id | transaction_id | account | type | amount | created_at |
|---|---|---|---|---|---|
| 1 | TXN-1234 | Current | DEBIT | 100 | 2026-07-02 |
| 2 | TXN-1234 | Savings | CREDIT | 100 | 2026-07-02 |

- Amount is always positive — the type column (DEBIT/CREDIT) tells you the direction
- Both entries share the same transaction_id to link them together
- Each entry has its own unique id

### Q: What happens if the debit write succeeds but the credit write fails?
Wrap both inserts in a single database transaction. If either fails, both roll back. This is atomicity — the A in ACID.

```sql
BEGIN TRANSACTION
  INSERT INTO ledger (transaction_id, account, type, amount) VALUES ('TXN-1234', 'Current', 'DEBIT', 100)
  INSERT INTO ledger (transaction_id, account, type, amount) VALUES ('TXN-1234', 'Savings', 'CREDIT', 100)
COMMIT
```

### Q: How do you scale the ledger when it has 5 billion rows?
1. **Read replicas** — direct all reads to replica, writes to primary. Reduces read load on primary.
2. **Sharding** — when the write primary itself becomes a bottleneck. Shard key = `account_id`.
   - Use `account_id` not `user_id` — most queries are by account, distributes more evenly
   - `user_id` creates hot spots for high-value customers

### Q: Sharding breaks ACID — debit on shard 1, credit on shard 2. How do you handle this?
Use the **Saga Pattern**.

---

## Topic 2: Saga Pattern

### What is it?
A Saga is a sequence of local transactions where each step has a compensating transaction that undoes it if something goes wrong.

```
Step 1: Debit £100 from Current Account (Shard 1)  → success
Step 2: Credit £100 to Savings Account (Shard 2)   → success → DONE

If Step 2 fails:
Compensating Step 1: Credit £100 back to Current Account (Shard 1)
```

### Key Point
Saga does not guarantee ACID. It gives eventual consistency with compensating transactions. Money may be in an inconsistent state for milliseconds while the saga completes — that is acceptable as long as the saga always completes either forward or backward.

### Two Types

**1. Choreography (event-driven)**
- Services communicate via events on a message broker (Kafka)
- Each service publishes a new event on success or failure
- Another service listens and triggers the next step or compensation
- No central coordinator — loosely coupled
- This is what TM's Vault Core uses

```
Payment Service → publishes DebitCompleted
    ↓
Savings Service listens → credits account → publishes CreditCompleted
    ↓
If fails → publishes CreditFailed
    ↓
Payment Service listens → reverses the debit (publishes DebitReversed)
```

**2. Orchestration (central coordinator)**
- One orchestrator service tells each service what to do
- Handles failures centrally
- Easier to reason about, harder to scale

### Important Distinction
On failure, the service publishes a **new failure event** — it does not put the original event back in the broker.

---

## Topic 3: Idempotency

### What is it?
If a payment request is sent twice, only one payment happens. Every financial API must be idempotent.

### The Problem
POST requests are not idempotent by default. Network failure after processing but before response → client retries → duplicate payment.

### Solution: Idempotency Key + Redis Cache

```
Request arrives with idempotency_key (transaction_id)
    ↓
Check Redis — key exists?
    ↓ YES                          ↓ NO
Return original cached result   Process transaction
                                Store result in Redis with key + TTL
                                Return result to client
```

### Key Design Decisions
- Idempotency key = `transaction_id` alone (not combined with amount — a retry with wrong amount should still be rejected by ID)
- Store the **original response** in Redis, not just a flag — client needs the result, not just "duplicate detected"
- TTL = 24 hours — prevents Redis growing forever, after TTL the request is treated as new
- Fallback: Redis → PostgreSQL if Redis is unavailable

### Flow for the network failure scenario
1. App sends POST /payments with transaction_id = TXN-999
2. Redis miss — not seen before
3. Server processes payment, writes to database
4. Server stores result in Redis with key TXN-999, TTL 24h
5. Network drops before response reaches app
6. App retries POST /payments with same transaction_id = TXN-999
7. Redis hit — already processed
8. Server returns original cached response
9. App receives success — no duplicate payment

---

## Topic 4: Event Sourcing

### What is it?
Store every event that led to the current state instead of storing the current state itself. Replay events to compute state at any point in time.

### Database Structure
```sql
SELECT account_id,
       SUM(CASE WHEN transaction_type = 'CREDIT' THEN amount
                WHEN transaction_type = 'DEBIT' THEN -amount
           END) AS balance
FROM ledger
WHERE account_id = 'ACC-001'
AND created_at <= '2024-03-15 15:47:00'
GROUP BY account_id
```

### Q: Replaying all events for 50M customers is too slow. How do you solve it?
Three approaches:

**1. Snapshot (most common)**
Checkpoint balance every N events. Replay only events after the last snapshot.
```
Snapshot at Event 50: Balance = £3,450
Event 51 to 75: replay only these
Current balance = £3,450 + events 51-75
```
Two tables: `ledger_events` (immutable, append-only) and `ledger_snapshots` (checkpoints)

**2. Materialised View / Read Model**
Pre-compute balance in a separate read table every time an event is written. No replay needed — just read the pre-computed value. Fastest reads. Risk: read model can get out of sync if update fails.

**3. Caching**
Cache the result after first replay. Fast but slightly stale.

| Approach | Use when |
|---|---|
| Snapshot | Need historical balances at any point in time |
| Materialised View | Only need current balance, speed is critical |
| Cache | Read heavy, slight staleness acceptable |

TM uses a combination — materialised view for current balance, full event log for audit and historical queries.

---

## Topic 5: CQRS

### What is it?
Separate writes (commands) from reads (queries) completely.

```
WRITE side (Command)          READ side (Query)
─────────────────             ─────────────────
Command Handler               Query Handler
    ↓                               ↓
Validates + processes         Read Model (pre-built)
    ↓                               ↓
Writes event to ledger        Returns result instantly
    ↓
Updates read model
```

### Why?
- Write side needs strong consistency and ACID — inherently slower
- Read side needs to be fast — users expect balance under 100ms
- Same database cannot be optimised for both

### Connection to Event Sourcing
Every event written on the command side triggers a read model update on the query side. Read model = pre-computed current state = materialised view.

### Q: User checks balance before read model is updated — sees old balance. How do you handle this?
- **Trade-off:** CQRS introduces eventual consistency. The lag between write and read model update is milliseconds in practice (Kafka consumer is near real-time)
- **UX solution:** Show "pending" state immediately after transaction so user knows it was received
- **In the interview:** Quantify the delay — "milliseconds, not seconds" and explain the pending state pattern

```
User deposits £500 → Write side confirms → UI shows "£500 deposit pending"
    ↓ milliseconds later
Read model updates → UI shows "Balance £1500"
```

---

## Topic 6: Outbox Pattern

### What is it?
Write the event to an outbox table in the same database transaction as the main write. A worker polls the outbox and publishes to Kafka. Guarantees no event is lost even if Kafka is down.

### The Problem it Solves
```
Write to database → SUCCESS
Publish to Kafka  → FAIL (Kafka down)
```
Downstream services never know the payment happened.

### Solution
```sql
BEGIN TRANSACTION
  INSERT INTO ledger (transaction_id, account, type, amount)
  INSERT INTO outbox (event_type, payload, status) VALUES ('PaymentProcessed', {...}, 'PENDING')
COMMIT
```
If either fails, both roll back. You can never have a payment without an outbox entry.

### Worker Flow
```
Worker polls outbox for PENDING records
    ↓
Mark as PROCESSING
    ↓
Publish to Kafka
    ↓
Mark as PROCESSED
```

### Q: Worker crashes after publishing but before marking PROCESSED. Kafka receives event twice. How do you handle it?
Make the **consumer idempotent** — you cannot always guarantee exactly-once delivery from the producer side.

```
Consumer receives event
    ↓
Check event_id — already processed?
    ↓ YES          ↓ NO
Skip it        Process + record event_id
```

At-least-once delivery + idempotent consumer = effectively exactly-once processing.

### Statuses
- PENDING — not yet published
- PROCESSING — picked up by worker, may have been published
- PROCESSED — confirmed published

---

## Topic 7: CAP Theorem — Banking Context

### The Three Properties
- **C — Consistency** — every read gets the most recent write
- **A — Availability** — every request gets a response
- **P — Partition Tolerance** — system keeps working during network splits

Network partitions always happen in distributed systems — P is not optional. The real choice is **CP or AP**.

### Banking Chooses CP
Banking systems choose Consistency over Availability. If there is a network partition, reject the request rather than process on stale data. Money cannot be lost or duplicated. Temporary unavailability is acceptable. Incorrect balance is not.

> "If the network link between London and Dublin goes down, we return an error to the user. We do not process a £500 transfer on potentially stale Dublin data. The user retries when the partition heals."

### Q: How do you detect the system is unhealthy and how quickly?
Layered observability — you need all four:

**1. Metrics — detect fast**
- Payment success rate drops from 99.9% to 0% — alert fires within 60 seconds
- Error rate spike on `/payments` endpoint
- Database connection pool exhaustion

**2. Distributed Tracing — find where it broke**
- Every request has a trace ID that follows it through every service
- See exactly which service in the chain failed and why
- Without tracing you are guessing which of 50 microservices dropped the request

**3. Structured Logging — understand what happened**
- Every log line has `trace_id`, `account_id`, `transaction_id`, `error_code`
- Query logs to find all failed payments during the outage window
- Needed for reconciliation — which customers need to be notified?

**4. Alerting — wake someone up**
- PagerDuty or OpsGenie, not just Slack
- Slack is for visibility, PagerDuty wakes up on-call engineer
- Alert threshold: payment error rate > 1% for 60 seconds

> "Metrics tell you something is wrong, tracing tells you where, logging tells you what happened to each request, and alerting wakes someone up. You need all four."

---

## Topic 8: API Design for Financial Systems

### Q: What makes POST /payments production-ready for a financial system?

**Authentication and Transport**
- JWT in Authorization header — only authorised users can call the endpoint
- HTTPS only — encryption in transit out of the box

**Input Validation**
- Validate all fields on arrival — fail fast with correct HTTP status codes
- 400 Bad Request for missing fields, 422 Unprocessable Entity for invalid values

**Idempotency**
- Idempotency key goes in the request header, not the body — this is industry standard (Stripe does this)
- Redis stores the key with 24h TTL — returns original response on duplicate

```
POST /v1/payments
Headers:
  Idempotency-Key: uuid-1234
  Authorization: Bearer <jwt>
```

**Rate Limiting**
- JWT tells you who you are. Rate limiting tells you how many times per minute.
- Without it, one bad client can take down the service
- Enforced at API Gateway level — e.g. 100 requests per user per minute

**Response Design**
- Return `202 Accepted` not `200 OK` — payment is queued, not instantly complete
- Response body includes `payment_id` and `status: PENDING`

**API Versioning**
- `/v1/payments` — change the contract later without breaking existing clients

**Audit Logging**
- Every payment attempt logged with `user_id`, `timestamp`, `ip_address`, `transaction_id`
- Required for compliance and fraud investigation

> "Auth, HTTPS, validation, idempotency key in header, rate limiting, 202 response with pending state, versioned endpoint, and audit log."

---

## Topic 9: Observability

### Q: Payment success rate dropped from 99.9% to 60% at 3am. Walk me through your investigation.

**The correct order:**

**1. Declare P1 incident**
- Create incident ticket immediately
- Notify stakeholders — "P1 incident logged, investigating"
- At 60% against 99.9% SLO — this is a breach, not a warning

**2. Check recent deployments**
- Quick sanity check — did anything go out in the last hour?
- If yes, rollback is the fastest fix

**3. Check metrics dashboard**
- Which service is failing? Since when?
- Error rate by endpoint — is it all payments or one specific flow?
- Database connection pool, Kafka consumer lag, third party provider latency

**4. Pull distributed traces for failed payments**
- Filter by `error_code` — which errors are spiking?
- Follow a single failed payment end to end using `trace_id`
- Find exactly which service in the chain is dropping requests

**5. Check downstream dependencies**
- Database — is it responding?
- Kafka — is the consumer processing messages or is lag growing?
- Third party providers — are their APIs returning errors?

**6. Check dead letter queue volume**
- Are failed messages piling up in the dead letter queue?
- Volume spike = messages being rejected, not just slow

**7. Fix or rollback**
- Simple fix: deploy hotfix, ask lead to prioritise
- Complex fix: rollback deployment, investigate root cause in business hours

**8. Close incident**
- Update ticket with findings and fix applied
- Schedule post-incident review

> "Metrics tell you something is wrong, tracing tells you where, logging tells you what happened to each request. Check deployments first, then metrics, then traces, then dependencies."

---

## Common Interview Follow-Up Questions

- What is the sharding key for a ledger and why?
- How do you prevent duplicate payments?
- What happens when Redis is down?
- How do you replay events to reconstruct a balance?
- What breaks first at 10x scale?
- How do you handle a saga that is stuck halfway?
- What is the difference between choreography and orchestration?
- Why does banking choose consistency over availability?
