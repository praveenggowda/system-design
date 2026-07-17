# Banking Ledger — Interview Q&A

---

## Clarifications

- Operations: POST /deposit, POST /withdrawal, POST /transfer, GET /balance
- Every transaction is double-entry — one debit entry and one credit entry, committed atomically
- Balance check: real-time, under 100ms
- Writes: can show "pending" to the user but the system must be strongly consistent — no stale balance for correctness decisions
- Idempotency required — same request must not create duplicate transactions
- Audit trail required — 7 years retention for regulatory compliance
- Single currency for now, design to allow multi-currency later
- Auth handled by a separate identity service — not in scope

---

## Back of Envelope

Use the 100K seconds shortcut. State it out loud and move on.

```
Peak:    2,000 TPS × 100K = 200M transactions/day  (infrastructure sizing)
Average: 500 TPS × 100K   = 50M transactions/day   (storage sizing)

Storage: 2KB × 50M = 100GB/day
         100GB × 365 ≈ 35TB/year
         35TB × 7 years = 250TB
         × 3 replicas = ~750TB → tier: 90 days hot, rest cold (S3 Glacier)

Write throughput: 2KB × 2,000 = 4MB/s → single PostgreSQL primary handles this
```

Say: "I will tier storage — 90 days hot in PostgreSQL, everything older moves to cold storage. 4MB/s write throughput is well within what a sharded PostgreSQL cluster handles."

---

## High-Level Architecture

```
Users
  ↓
Load Balancer
  ↓
API Gateway (auth, rate limiting)
  ↓
  ├── GET /balance → Balance API Service
  │         ↓
  │       Redis Cache
  │         ↓ (cache miss)
  │       Read Replica → Pre-Computed Balance Table
  │
  └── POST /deposit|withdrawal|transfer → Ledger API Service
              ↓
         Idempotency check (transaction_id → Redis)
              ↓
         PostgreSQL Primary (same shard)
         BEGIN TRANSACTION
           INSERT INTO ledger_entries (debit)
           INSERT INTO ledger_entries (credit)
           UPDATE account_balances
           INSERT INTO outbox
         COMMIT
              ↓
         Return 200 to user
              ↓
         Outbox Publisher (polls outbox)
              ↓
             SNS
         ┌────┴────────────┬──────────────────┐
   Notification SQS   Balance Projection   Reconciliation
   Consumer           Consumer             Consumer
   (notify user)      (update read replica  (global balance
                       balance table)        invariant check)
```

Cross-shard transfer: Saga Coordinator handles debit on shard A, then credit on shard B, with compensating rollback if credit fails.

---

## CQRS — Read and Write Separation

Two separate services:

**Balance API Service (read path)**
- Checks Redis first
- Cache miss → pre-computed balance table on Read Replica
- Never queries the primary
- Never SUMs raw ledger entries on every request (O(n) — never do this)

**Ledger API Service (write path)**
- Handles all state-changing operations
- Writes synchronously to PostgreSQL Primary
- No message queue in the critical write path (see below)

---

## Why No Queue in the Critical Write Path

A queue between the API and the database introduces a window where the balance is stale.

If the user withdraws £500 and the message sits in the queue unprocessed, the balance in PostgreSQL still shows £600. A second withdrawal request passes the balance check and both are processed.

**Correct pattern:**
- Write synchronously to PostgreSQL (ledger entries + balance update + outbox event in one transaction)
- Return 200 to the user immediately
- Fan out to downstream services (notification, audit, analytics) asynchronously via Outbox

Before using a queue, ask: "Does the caller need the result of this operation before they can continue?" If yes → synchronous. If no → queue.

---

## Double-Entry Bookkeeping

Every transaction produces exactly two ledger entries. The sum of all credits minus all debits across the entire system must always equal zero. If it does not, money was created or destroyed — this must page someone immediately.

```sql
ledger_entries (
  entry_id        UUID PRIMARY KEY,
  transaction_id  UUID NOT NULL,       -- idempotency key
  account_id      UUID NOT NULL,
  amount          DECIMAL NOT NULL,    -- positive for credit, negative for debit
  type            VARCHAR NOT NULL,    -- DEBIT or CREDIT
  created_at      TIMESTAMPTZ NOT NULL
)
```

Ledger entries are **immutable and append-only**. Never UPDATE a ledger entry. Only INSERT.

---

## Pre-Computed Balance Table

Never calculate balance by SUMming all ledger entries on every GET /balance request. At 7 years of history, that is billions of rows.

```sql
account_balances (
  account_id   UUID PRIMARY KEY,
  balance      DECIMAL NOT NULL,
  updated_at   TIMESTAMPTZ NOT NULL
)
```

The Balance Projection Consumer updates this table every time a transaction is committed via the Outbox. The pre-computed table lives on the Read Replica and is the source for all balance reads.

---

## Idempotency

Two levels:

**1. At the API level (Redis)**
Check the transaction_id in Redis before processing. If seen before, return the original response.
TTL: 24 hours.

**2. At the database level**
Before inserting ledger entries, check if a record with that transaction_id already exists. If yes, skip. The UNIQUE constraint on transaction_id enforces this.

```sql
UNIQUE(transaction_id, account_id, type)
```

Redis is the fast path. The database constraint is the safety net.

---

## Outbox Pattern

Dual-write problem: Ledger API writes to PostgreSQL, then crashes before publishing to SNS. Downstream services miss the event.

Solution: write ledger entries, balance update, and outbox event atomically in one transaction.

```sql
BEGIN TRANSACTION
  INSERT INTO ledger_entries ... (debit)
  INSERT INTO ledger_entries ... (credit)
  UPDATE account_balances SET balance = balance - amount WHERE account_id = ?
  INSERT INTO outbox (event_type, payload)
COMMIT
```

Outbox Publisher polls the outbox table and publishes to SNS. If SNS is down, events stay in the outbox and are retried. No event is ever lost.

Rule: whenever a message queue is in a financial system, the interviewer will ask "what if the queue is down?" The Outbox pattern is always the answer.

---

## Pessimistic Locking

Use pessimistic locking (SELECT FOR UPDATE) for all balance-affecting writes. Optimistic locking forces retries on conflict, which is not acceptable in banking.

```sql
BEGIN;

SELECT balance FROM account_balances
WHERE account_id = 'xyz' FOR UPDATE;

-- application checks: if balance < withdrawal amount → ROLLBACK

INSERT INTO ledger_entries (transaction_id, account_id, amount, type)
VALUES ('txn-123', 'xyz', 500, 'DEBIT');

UPDATE account_balances
SET balance = balance - 500
WHERE account_id = 'xyz';

COMMIT;
```

First transaction to acquire the lock wins. Second transaction waits, then re-reads the updated balance.

---

## Deadlock Prevention — Lock Ordering

**Problem:** Transfer A → B and Transfer B → A running simultaneously.
- Worker 1: locks A, waits for B
- Worker 2: locks B, waits for A
- Deadlock.

**Solution:** Always lock in ascending account_id order.

```
Transfer A (id=100) → B (id=200): lock 100 first, then 200
Transfer B (id=200) → A (id=100): lock 100 first, then 200
```

Both workers always lock the same account first. No circular wait. No deadlock.

Set a lock timeout as a safety net — if a lock is not acquired within a few seconds, roll back and retry.

---

## Sharding

At high write throughput, a single PostgreSQL primary becomes the bottleneck. Shard by account_id.

**Same-shard transfer:** both accounts on the same shard. Single ACID transaction. Lock ordering applies.

**Cross-shard transfer:** accounts on different shards. Cannot use a single ACID transaction. Use the Saga pattern:

```
Saga Coordinator:
  1. Debit shard A (write debit entry + mark as PENDING)
  2. On success → Credit shard B (write credit entry)
  3. On failure → Compensating transaction: reverse the debit on shard A
```

The Saga Coordinator tracks state. If any step fails, compensating transactions restore consistency.

---

## Redis Cache — Write-Through, Not Write-Back

Cache the pre-computed balance in Redis for fast reads.

**Write-through:** write to the database first, then update the cache. Cache is a read optimisation, not the source of truth.

**Write-back is dangerous for financial data.** Write-back stores the balance in cache and flushes to the database later. If the cache node dies before flushing, the balance update is lost permanently.

Always write-through for financial data. The database is always authoritative.

Cache TTL: 60 seconds. Trade-off: user may see a balance up to 60 seconds stale on read. Acceptable for display. The write path always uses the strongly-consistent database, never the cache.

---

## Tiered Storage

7 years of transaction history is expensive to keep in hot PostgreSQL.

Strategy:
- 0 to 90 days: hot storage — PostgreSQL Primary + Read Replicas (fast query)
- 90 days to 2 years: warm storage — PostgreSQL with cheaper storage class
- 2 years to 7 years: cold storage — S3 Glacier (infrequent access, compliance archive)

A separate archival job runs nightly to move aged records to cold storage.

---

## Failure Scenarios

| Failure | Behaviour |
|---|---|
| Duplicate request (same transaction_id) | Redis idempotency check returns original response; DB UNIQUE constraint as safety net |
| Write to DB succeeds, SNS down | Outbox retains event; Publisher retries when SNS recovers |
| Outbox Publisher crashes | Another instance picks up unpublished events |
| Cache miss (Redis down) | Fall back to Read Replica pre-computed balance table |
| Cross-shard transfer partial failure | Saga Coordinator triggers compensating debit reversal on shard A |
| Deadlock on concurrent transfers | Lock ordering prevents circular wait; lock timeout triggers retry |
| Read Replica replication lag | Balance may be slightly stale for reads; write path always hits primary |

---

## Observability

| Signal | What to monitor |
|---|---|
| Transaction throughput | Drop = upstream failure or traffic anomaly |
| Write latency (p50/p95/p99) | Spikes = lock contention or DB saturation |
| Balance check latency | Must stay under 100ms — alert at p99 > 80ms |
| Outbox depth | Unpublished events accumulating = SNS or Publisher down |
| DLQ depth > 0 | Failed downstream processing — immediate alert |
| Redis hit/miss ratio | Low hit rate = cache not warming or TTL too short |
| Replication lag | Read replica falling behind — balance reads may be stale |
| **Global ledger balance** | Sum of all credits minus all debits must equal zero. If not, money was created or destroyed. Page immediately. |

Structured logs: every log entry includes transaction_id, account_id, amount, type, status, and trace_id.

Distributed trace: API → PostgreSQL → Outbox → SNS → downstream consumers.

---

## Trade-offs

| Decision | Why | Alternative |
|---|---|---|
| PostgreSQL over NoSQL | ACID, relational model, regulatory compliance, 7-year retention | DynamoDB — scales but lacks multi-row ACID and complex queries |
| Pre-computed balance table | O(1) balance reads, no SUM() on billions of rows | Sum ledger entries on each request — O(n), not viable at scale |
| Pessimistic locking | Banking cannot afford conflicts and retries | Optimistic locking — retry on conflict, unacceptable in financial context |
| Write-through cache | Database always authoritative, no data loss risk | Write-back — faster but catastrophic if cache dies before flush |
| Outbox pattern | No event loss on crash between DB write and SNS publish | Direct publish — event lost if process crashes mid-flight |
| Sharding by account_id | Even distribution, scales writes | Single primary — bottleneck at high TPS |
| Saga for cross-shard | Only option for distributed ACID-like guarantee | 2PC — exists but locks across shards, poor performance at scale |
| Synchronous write path | Consistent balance, no stale-read window | Queue in critical path — stale balance allows double spend |

---

## Diagram

See `banking-ledger.drawio`.

Fix: the "write back balance" label from Read Replica to Redis should read "write-through" or "update cache" — write-back is the dangerous pattern.
