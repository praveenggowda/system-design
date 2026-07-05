# Banking Ledger System — Reference Design

> Reference material for study. Use this to compare against your own design after you build it.

---

## Swim Lane Structure

**Layers:**
- Client Layer
- API Layer
- Queue / Asynchronous Processing Layer
- Data Layer

---

## Write Path — POST /payment

```
Client
  ↓
Load Balancer
  ↓
API Gateway (auth, rate limiting)
  ↓
Payment / Ledger API Service
  ↓
PostgreSQL Primary Shard

Within one atomic database transaction:
  1. Validate request
  2. Check idempotency key
  3. Check available balance
  4. Write immutable debit and credit ledger entries
  5. Update account balance table
  6. Store idempotency result
  7. Write outbox event
  8. Commit transaction

  ↓
Outbox Publisher polls outbox
  ↓
Event Queue / Stream (Kafka or SQS)
  ↓
Consumers: Notification / Reconciliation / Analytics
```

---

## Read Path — GET /balance

```
Client → Load Balancer → API Gateway → Balance API Service → Redis Cache

Cache hit:
  Return balance

Cache miss:
  PostgreSQL Read Replica → pre-computed balance table → store in Redis → return balance
```

---

## Cross-Shard Payment Flow

```
Same shard:
  One atomic ACID transaction

Different shards:
  Create durable transfer record
  Saga Coordinator
    → Debit source account
    → Credit destination account
    → Mark transfer completed

  On failure:
    Retry idempotently
    Reconcile stuck transfers
    Compensate only when appropriate
```

---

## Database Replication

```
PostgreSQL Primary Shards → PostgreSQL Read Replicas
Event Queue → Balance Projection Consumer → PostgreSQL Read Replica (balance table)
```

---

## Key Design Principles

1. The immutable ledger is the source of truth
2. The account balance table is a materialised view for fast reads
3. Redis and read replicas may be slightly behind primary — acceptable for reads, not for write authorisation
4. Payment authorisation and balance validation must use the primary shard
5. Every payment request must carry an idempotency key
6. Outbox pattern ensures a committed ledger transaction never loses its downstream event
7. Shard by account_id — stable, high-cardinality, keeps account data co-located
8. Cross-shard transfers require Saga Coordinator, durable states, idempotent steps, reconciliation
9. Use Decimal or integer minor units for money — never float
10. Monitor: payment success rate, failed payments, duplicate requests, pending transfers, replica lag, cache hit rate, queue depth, balance projection lag, reconciliation mismatches
