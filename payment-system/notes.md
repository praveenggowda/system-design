# Payment Platform — Reference Design

> Reference material for study. Use this to compare against your own diagram after you build it.

---

## Scope

- Accept payment instructions
- Validate payer, payee, balance, limits, and payment details
- Perform fraud, AML, and sanctions checks
- Route payments to the correct payment rail
- Track payment status
- Prevent duplicate payments
- Notify customers
- Reconcile payment status with external payment rails

---

## APIs

### POST /payments
```
Headers:
  Idempotency-Key: <UUID>

Request:
{
  "source_account_id": "A100",
  "destination_account": {
    "sort_code": "12-34-56",
    "account_number": "12345678"
  },
  "amount_minor": 5000,
  "currency": "GBP",
  "reference": "Rent"
}

Response:
{
  "payment_id": "pay_123",
  "status": "PENDING"
}
```

### Other APIs
- GET /payments/{payment_id}
- GET /accounts/{account_id}/payments
- POST /payments/{payment_id}/cancel

**Money rule: always use integer minor units. £50.00 = 5000 pence. Never use float.**

---

## Architecture Layers

### Client Layer
- Mobile App, Web App, External API Client

### Edge / API Layer
- Load Balancer, API Gateway, Auth/Authz, Rate Limiting, Payment API Service

### Core Payment Layer
- Idempotency Service
- Payment Orchestrator
- Validation Service
- Limits Service
- Fraud / AML / Sanctions Service
- Payment State Machine
- Payment Router
- Rail Connectors: Faster Payments, CHAPS, Bacs, SWIFT

### Data Layer
- Payment Database
- Account Ledger / Core Banking
- Idempotency Records
- Outbox Table
- Audit Store
- Reconciliation Store

### Async Layer
- Outbox Publisher
- Event Queue / Stream
- Notification Consumer
- Statement Consumer
- Reconciliation Consumer
- Reporting / Analytics Consumer

### External Systems
- Payment Rails, Receiving Banks, Fraud/AML Providers, Sanctions Providers

---

## Payment Status State Machine

```
CREATED
  ↓
VALIDATED
  ↓
PENDING_COMPLIANCE
  ↓
REJECTED (if compliance fails)

FUNDS_RESERVED
  ↓
SUBMITTED
  ↓
ACCEPTED_BY_RAIL
  ↓
SETTLED

FAILED
REVERSED
```

---

## Payment Flow

```
POST /payments with Idempotency-Key
  ↓
API Gateway authenticates and rate-limits
  ↓
Check idempotency record
  ↓
Validate payment request
  ↓
Validate account status
  ↓
Validate beneficiary details
  ↓
Validate payment limits
  ↓
Check available balance
  ↓
Run fraud / AML / sanctions checks
  ↓
Create payment record — PENDING
  ↓
Reserve funds or debit account
  ↓
Select payment rail
  ↓
Send payment instruction via rail connector
  ↓
Receive acknowledgement / callback / settlement
  ↓
Update payment status
  ↓
Write outbox event → publish to queue
  ↓
Notify customer, update statement, reconcile
```

---

## Idempotency

Store per key: `idempotency_key`, `client_id`, `request_hash`, `payment_id`, `response`, `status`, `created_at`

Rules:
- Same key + same request → return original result
- Same key + different request → reject as conflict
- New payment → new idempotency key

---

## Ledger and Money Movement

Immutable double-entry ledger. Sum of all entries must equal zero. Never edit or delete a ledger entry — create a reversal entry instead.

---

## Scaling

- Peak TPS: 10,000 / Average TPS: 1,000
- 86.4M payments/day → 172.8M ledger entries/day
- Shard by account_id — high-cardinality, stable, keeps account data together
- Same shard → single ACID transaction
- Cross shard → durable Saga with idempotent steps and reconciliation

---

## Failure Handling

| Failure | Response |
|---|---|
| Client timeout | Retry with same idempotency key |
| Fraud provider unavailable | Fail closed or queue for review |
| Payment rail timeout | Keep SUBMITTED, reconcile using payment reference |
| Duplicate queue message | Consumer checks event ID, process once |
| Outbox publish fails | Outbox publisher retries |
| External payment fails after funds reserved | Release reservation, create reversal, notify |
| Cross-shard transfer fails | Saga retry, reconcile, compensate |

---

## Observability

Monitor: payment success rate, failure rate, latency p50/p95/p99, queue depth, connector error rate, fraud/AML decision latency, pending payment age, duplicate idempotency requests, reconciliation mismatches, ledger imbalance, replication lag, saga failures, stuck transfers.

---

## Key Interview Sentences

> "A payment is a long-running state machine. I persist every state transition durably and make every step idempotent."

> "The immutable double-entry ledger is the source of truth. Payment status and balances are derived or materialised views."

> "I use idempotency keys to make client retries safe and prevent duplicate debits."

> "I do not assume a payment rail timeout means failure. I keep the payment pending and reconcile using a payment reference."

> "For a single account, I keep balance validation and ledger posting strongly consistent. For cross-shard transfers, I use a durable Saga with idempotent steps and reconciliation."

> "The synchronous path should be short: authenticate, validate, persist durable payment state, and acknowledge. Everything else is async."
