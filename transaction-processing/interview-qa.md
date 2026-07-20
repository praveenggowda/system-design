# Transaction Processing System — Interview Q&A

---

## Clarifications

- Transactions come from two sources: user-initiated via UI, and upstream services (payment gateways, batch processors, standing orders, direct API integrations)
- Transaction types in scope: debit, credit, transfer between accounts
- Idempotency required: callers provide a transaction_id; the system must guarantee at-most-once financial effect
- Consistency over availability: CP system — a transaction must never be lost or applied twice
- Synchronous acknowledgment to the caller in under 200ms; downstream processing is async
- Data retention: 7 years minimum (regulatory requirement)
- Real-time notifications: out of scope — handled by a separate notification service downstream
- Auth: handled upstream, not in scope
- Fraud detection: in scope as a synchronous validation step in the write path

---

## How This Differs From Payment System and Banking Ledger

Three systems, three distinct responsibilities:

**Payment System** — external facing. Talks to PSPs and payment rails. Manages the payment lifecycle (PENDING → PROCESSING → SETTLED). Owns retries, Saga, idempotency against external systems. Produces a settled event.

**Transaction Processing System** — the internal engine. Receives settled events or direct requests. Validates: sufficient funds, duplicate check, limits, fraud. Atomically writes ledger entries and materialized balances. Publishes a transaction-posted event. Owns the core business logic of WHAT gets recorded and WHY.

**Banking Ledger / Reporting** — downstream consumers of posted events. Serve read queries, projections, statements, audit reporting. Do not own the authoritative write.

---

## Back of Envelope

```
Peak:    1000 TPS × 100K seconds = 100M transactions/day
Average: 500 TPS × 100K seconds  = 50M transactions/day

Storage per transaction: ~2KB
50M × 2KB = 100 GB/day

100 GB × 365 = 36.5 TB/year

Retention strategy:
  Hot  (0-2 years):  PostgreSQL — ~73 TB with 3 replicas = ~220 TB
  Warm (2-7 years):  Object storage (S3, encrypted, immutable)
                     36.5 TB × 5 years = ~182 TB — much cheaper

Do NOT assume all 7 years sit in PostgreSQL with 3 replicas.
That would be ~768 TB of expensive operational storage.
Use tiered storage.

Throughput: 2KB × 1000 TPS = 2 MB/s ingestion bandwidth

Accounts: 50M total, 10M active (20%)
```

---

## Balance Model — Two Layers

Define this clearly upfront. Two layers, both maintained:

**Immutable Ledger Entries** — authoritative financial history. Never updated, never deleted. Every debit and credit is appended forever.

**Materialized Account Balance** — fast current balance for reads, validation (sufficient funds), and overdraft prevention.

Both are updated atomically in the same DB transaction. The ledger is the source of truth. The materialized balance is a pre-computed projection of it.

A reconciliation job periodically verifies: `materialized_balance = SUM(ledger_entries)`. Any mismatch triggers an alert.

---

## Ledger Ownership — Choose One Model

Do not have TPS and a downstream Ledger System independently owning the authoritative ledger. Pick one:

**Option 1 (chosen for this design): TPS owns the ledger directly**

TPS atomically writes:
- Transaction record
- Debit ledger entry
- Credit ledger entry
- Materialized balance updates
- Outbox event

All in one DB transaction. Downstream consumers (reporting, audit, notifications) receive events via Outbox → SNS. They do NOT write the authoritative ledger — they build projections and reports from it.

**Option 2: Separate Ledger Service**
TPS validates and orchestrates. Ledger Service owns the atomic write. TPS calls Ledger Service synchronously. Cleaner separation but adds network hop and availability dependency.

For this design: Option 1. TPS and ledger are the same transactional boundary.

---

## High-Level Architecture

```
Two ingestion paths:

UI / External API callers
  → API Gateway (auth, rate limiting, routing)
  → Transaction Processing Service

Upstream services (batch, standing orders, internal systems)
  → SQS Queue
  → Transaction Worker (polls queue)
  → Same Transaction Processing logic

Transaction Processing Service
  1. Idempotency check (Redis — fast path)
  2. Validation (account exists, sufficient funds — from materialized balance)
  3. Fraud check (gRPC — strict timeout + circuit breaker)
  4. Atomic DB write (PostgreSQL Primary)
  5. Return 200 to caller

Outbox Worker
  → SNS fan-out
  → Audit queue, Notification queue, Reporting queue, Analytics queue

PostgreSQL Primary
  Tables: transactions (UNIQUE transaction_id), ledger_entries,
          account_balances (materialized), outbox
```

---

## Idempotency — Two-Layer Guarantee

Redis is a fast-path optimisation. PostgreSQL UNIQUE constraint is the correctness guarantee.

**Why Redis alone is not enough:**
Two identical requests can arrive simultaneously. Both check Redis. Both get a MISS. Both proceed to process. Without a DB-level constraint, the transaction could be applied twice.

**Correct approach:**

```
Request arrives with transaction_id

Step 1: Redis GET idempotency:{transaction_id}
  → HIT:  return cached response immediately (no DB touched)
  → MISS: continue

Step 2: Inside the DB transaction:
  BEGIN
    INSERT INTO transactions (transaction_id, ...)
    -- UNIQUE constraint on transaction_id prevents duplicate
    -- If duplicate: constraint violation → rollback → return cached/idempotent response
    UPDATE account_balances ...
    INSERT INTO ledger_entries (DEBIT) ...
    INSERT INTO ledger_entries (CREDIT) ...
    INSERT INTO outbox ...
  COMMIT

Step 3: After commit: SET Redis idempotency:{transaction_id} = result (TTL: 24h)
```

Interview line: "Redis provides a fast-path idempotency check to avoid unnecessary processing, but the actual correctness guarantee is enforced by a UNIQUE constraint on transaction_id in PostgreSQL. Even if two identical requests pass the Redis check concurrently, only one can commit."

---

## Write Path — One Transaction End to End

```
1. Caller submits POST /transactions
   Body: { transaction_id, from_account, to_account, amount, type }

2. API Gateway: auth, rate limiting

3. Transaction Processing Service:

   a. Redis idempotency check — HIT: return cached. MISS: continue.

   b. Load account state (materialized balance, current version)

   c. Validation: account exists? sufficient funds? within limits?

   d. Fraud check (gRPC, strict timeout)
      → Fraud Service unavailable: fail closed (reject transaction)
      → Takes too long: circuit breaker trips, fail closed

   e. Atomic DB write (single transaction):
      BEGIN
        INSERT INTO transactions (transaction_id, ...)   ← UNIQUE enforces idempotency
        UPDATE account_balances SET balance = balance - 100, version = version + 1
          WHERE id = from_account AND version = ?        ← optimistic locking
        UPDATE account_balances SET balance = balance + 100, version = version + 1
          WHERE id = to_account AND version = ?
        INSERT INTO ledger_entries (from_account, -100, DEBIT)
        INSERT INTO ledger_entries (to_account, +100, CREDIT)
        INSERT INTO outbox (event_type, payload)
      COMMIT

   f. Write idempotency result to Redis

   g. Return 200 to caller

4. Outbox Worker polls → publishes to SNS → marks processed

5. SNS fan-out:
   → Audit queue
   → Notification queue
   → Reporting / analytics queue
```

---

## Concurrency — Optimistic vs Pessimistic Locking

Both are valid depending on contention. Do not say "never use pessimistic locking."

**Optimistic locking** — use when contention is low:
```sql
UPDATE account_balances
SET balance = balance - 100, version = version + 1
WHERE id = 'account-x' AND version = 4 AND balance >= 100
```
- rows_affected = 1 → success
- rows_affected = 0 → conflict → TPS reloads latest state → retry or return insufficient funds

**Pessimistic locking** — use when contention is high:
```sql
SELECT * FROM account_balances WHERE id = 'account-x' FOR UPDATE
```
Locks the row until commit. Better when hot accounts receive many concurrent writes and retry storms are expensive.

Interview framing: "I would start with optimistic concurrency control given low expected contention. If we observe high contention on hot accounts in production, pessimistic locking or account-level serialisation becomes more appropriate."

Version numbers are internal to the TPS. Callers never send them.

---

## At-Most-Once Financial Effect — Correct Framing

Do not say "at-most-once delivery." Infrastructure cannot guarantee this without losing messages.

Correct framing:
```
At-least-once delivery
+
Idempotent processing (UNIQUE transaction_id)
=
At-most-once financial effect
```

Retries are safe. The transaction_id UNIQUE constraint ensures only one financial effect regardless of how many times the request is delivered.

---

## Fraud Service in the 200ms Write Path

The fraud check is synchronous in the write path. Every ms matters.

Requirements:
- Strict timeout on the gRPC call (e.g. 50ms)
- Circuit breaker: if Fraud Service is slow or unavailable, trip the circuit
- Failure policy: fail closed for a financial system — if fraud cannot be assessed, reject the transaction. Caller retries safely using the same transaction_id.

Some fraud checks can be async (post-commit review, flag for manual review). High-risk synchronous checks that must block the transaction stay in the write path.

---

## Scaling Concurrency at 10x (10K TPS)

**Account-level in-memory queuing does not work across multiple TPS instances.**

If TPS is horizontally scaled to 10 instances, each instance has its own memory. They do not share account queues. Two instances can still process conflicting transactions against the same account simultaneously.

**Correct approach at scale: Kafka partitioning by account_id**

```
Transaction submitted
  → Partition key = account_id
  → Kafka routes to one partition
  → One consumer per partition processes in order

Transfer (Account A → Account B) complicates this:
  → A and B may be on different partitions
  → Requires careful partition strategy or accepting that the DB handles concurrency
```

At 1000 TPS: DB concurrency control (optimistic/pessimistic) is sufficient.
At 10K TPS: Kafka partitioning by account_id serialises writes per account without DB lock contention.

---

## Cross-Shard Transfers — Nuanced Answer

Sharding on account_id solves single-account write throughput. The hard problem: Account A (shard 1) and Account B (shard 2) must be updated atomically.

Options — do not automatically say "Saga is the standard solution":

**Two-Phase Commit (2PC)** — coordinator locks both shards. Complex, slow, coordinator is a single point of failure. Rarely used in modern systems.

**Saga pattern** — DEBIT shard 1, emit event, CREDIT shard 2. Compensate (reverse DEBIT) if shard 2 fails. Problem: there is a window where £100 is debited but not yet credited — temporary inconsistency in the authoritative ledger. Acceptable for some domains, problematic for others.

**Avoid sharding the problem** — design partition boundaries so that transfers between related accounts land on the same shard. Accounts within the same bank/product line co-located.

**Pending state model** — write DEBIT as PENDING, write CREDIT as PENDING, then confirm both. Explicit intermediate states, no temporary disappearance of funds.

Interview framing: "Cross-shard atomic ledger posting is a major design challenge. I would not automatically choose Saga without first clarifying consistency requirements. Options include co-locating related accounts on one shard, pending-state workflows, or distributed transaction protocols. The right choice depends on how much transient inconsistency the business can tolerate."

---

## Failure Scenarios

| Failure | Behaviour |
|---|---|
| Duplicate transaction | Redis fast-path returns cached result. If Redis miss, PostgreSQL UNIQUE constraint blocks duplicate commit. |
| TPS crashes after DB commit | Outbox event is committed. Worker picks it up on recovery. No event lost. |
| Outbox Worker fails to publish | Retry with exponential backoff. After N retries: mark dead, alert on-call. |
| Redis unavailable | Skip Redis fast-path. PostgreSQL UNIQUE constraint remains the correctness guarantee. |
| Validation Service unavailable | Return 503. Do not write. No partial state. |
| Fraud Service unavailable | Fail closed. Return temporary failure. Caller retries safely with same transaction_id. |
| Concurrent writes — same account | Optimistic locking: first writer wins, second detects conflict, retries or returns insufficient funds. |
| Cross-shard transfer failure mid-way | Depends on model chosen. Pending state: mark as failed, reverse pending debit. Saga: compensating transaction. |

---

## Tiered Storage — Seven Year Retention

Do not keep all 7 years in PostgreSQL with 3 replicas. That is expensive and unnecessary.

```
0 to 2 years:   Hot — PostgreSQL Primary + Read Replicas
                Fast queries, operational access

2 to 7 years:   Cold — S3 (Parquet, partitioned by date, encrypted, immutable)
                Queryable via Athena for regulatory lookups
                Cheap object storage, durable

Nightly archival job: moves records older than N days to S3,
marks archived in PostgreSQL.
```

All 7 years remain retrievable. Only recent operational data stays in PostgreSQL.

---

## Observability

**Logs**
Structured logs on every transaction: transaction_id, account_id (masked), amount, type, status, trace_id, timestamp. trace_id flows from API Gateway through every downstream system.

**Metrics**

| Metric | Why |
|---|---|
| Transaction throughput (TPS) | Drop = upstream issue or service degradation |
| Success vs failure rate by reason | Insufficient funds vs fraud vs duplicate — each means something different |
| Idempotency hit rate | Spike = upstream retrying aggressively |
| DB conflict / retry rate | Spike = high contention on hot accounts |
| Outbox depth | Growing = Worker behind or down |
| Outbox dead letter count | Must always be zero — any value is an immediate page |
| End-to-end write latency p99 | From request received to DB committed — must stay under 200ms |
| Fraud Service latency | External dependency in the critical write path |
| Validation Service latency | Same — slowdown blocks every transaction |
| Ledger reconciliation failures | Materialized balance diverged from ledger sum — critical alert |

**Traces**
Every transaction carries a trace_id from API Gateway through TPS → Validation → Fraud → DB → Outbox Worker → SNS → downstream consumers.

**Autoscaling**
Scale API/TPS services on CPU and request rate. Scale Outbox Worker on SQS queue depth.

---

## Trade-offs

| Decision | Why | Alternative | Cost |
|---|---|---|---|
| PostgreSQL UNIQUE + Redis fast-path | UNIQUE is the correctness guarantee. Redis is the performance optimisation. | Redis alone | Race condition: two concurrent requests both miss Redis and apply duplicate transaction |
| Materialized balance + immutable ledger | Fast reads, fast balance checks, full audit trail | Derive balance from ledger entries on every read | Expensive at scale — SUM over millions of entries per account per request |
| Optimistic locking at 1000 TPS | Low contention, non-blocking | Pessimistic locking | Better for high-contention hot accounts but serialises writes |
| SNS + SQS for fan-out | Simpler, managed | Kafka at 10x | Ordered processing per account, replay, higher throughput — operational complexity |
| Fail closed on Fraud Service unavailability | Financial system cannot allow unvalidated transactions | Fail open | Risk of fraud passing through when fraud service is down |
| Tiered storage for 7-year retention | Cost-efficient, operationally manageable | All data in PostgreSQL | Hundreds of TB of expensive operational DB storage for rarely-queried historical data |
| TPS owns ledger writes | Single transactional boundary, no dual-write | Separate Ledger Service | Cleaner separation of concerns but adds network hop and availability dependency |
