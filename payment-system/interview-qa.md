# Payment Platform — Interview Q&A

> Live mock interview session. Questions asked by interviewer, answers given and gaps filled.

---

## Requirements Gathering

**Q: Design a Payment Processing Platform.**

Clarifying questions asked:
- Is this credit card, loan, mortgage, or account-to-account transfer?
- How many TPS at peak?
- How many registered users and DAU?
- Domestic or international payments?
- What is the SLA for payment completion?

Answers given:
- Scope: account-to-account transfers (UK domestic)
- Peak TPS: 50,000
- Registered: 100M, DAU: 10M
- Domestic only — UK Faster Payments
- SLA: real-time, payment completes within 10 seconds

**Two questions missed during requirements gathering:**
1. Domestic or international? — huge design impact (SWIFT vs Faster Payments)
2. What is the payment SLA? — determines sync vs async processing

---

## Difference: Banking Ledger vs Payment Platform

- Banking Ledger — records what happened. Source of truth. Answers "what is my balance?"
- Payment Platform — orchestrates the movement of money. Handles validation, compliance, routing, failure. Writes to the ledger once confirmed.
- Payment Platform is upstream of the Ledger. They connect at the point where a confirmed payment creates a ledger entry.

---

## Functional Requirements

- Process payment from account A to account B
- Validate payment request — account exists, sufficient balance
- Check for duplicate payments (idempotency)
- Return payment status to the caller
- Write confirmed payment to the ledger

---

## Non-Functional Requirements

- Consistency over availability — CP
- SLA: payment completes within 10 seconds
- Idempotency — same payment processed only once
- Durability — zero data loss, at-least-once delivery
- Scale: 50,000 TPS peak, 100MB/sec write throughput
- Availability SLO: 99.99%
- Security — authentication, authorisation, fraud detection

---

## Back of Envelope

- Peak TPS: 50,000 / Average TPS: 5,000
- Daily volume: 5,000 * 86,400 = 432M transactions/day
- User sanity check: 432M / 10M DAU = 43 transactions per user per day
- Storage: 432M * 2KB (double-entry) = 864GB/day → 315TB/year → 945TB with 3 replicas
- Write throughput at peak: 50,000 * 2KB = 100MB/sec
- Implication: sharding required

**Gap corrected:** Original calculation wrote 4.3M instead of 432M — missed the full seconds in a day (86,400 not 86.4).

**Indian to Western conversion:**
- 1 Lakh = 0.1 Million
- 1 Crore = 10 Million
- Always convert to Million/Billion immediately in the interview

---

## High Level Design Q&A

**Q: Where does the payment request come from — upstream system or client directly?**
Client sends POST /payments directly. Request contains source_account_id, destination_account_id, amount, currency, idempotency_key in header.

**Q: Walk me through the full payment flow.**

Answer:
- Load Balancer → API Gateway (auth, rate limiting)
- Check idempotency key in Redis — if exists return cached response
- Fetch pre-computed balance from Read Replica — validate account active and balance sufficient
- If valid, publish message to SQS — return 202 Accepted + payment_id immediately
- Payment Worker polls SQS → passes to Saga Coordinator
- Saga Coordinator writes ledger entry first → gets confirmation → updates Payment Status to COMPLETED

**Q: Where exactly does idempotency check happen?**
Before publishing to the queue. Check Redis first. If Redis misses, fall back to Read Replica. Never let a duplicate reach the queue.

**Gap corrected:** Cache miss fallback should go to Read Replica, not Write Primary. Write Primary is for writes only.

**Q: What if the balance check passes but the Payment Worker fails before writing to the ledger?**
Saga Coordinator tracks state durably. If no confirmation from ledger within timeout, publishes compensation event. Reconciliation Consumer picks up stuck payments and retries or reverses.

**Q: Why write to the ledger first before updating the payment table?**
Ledger is the source of truth. If you write to the payment table first and the ledger write fails, payment table says money moved but no ledger entry exists — consistency violation. Ledger first guarantees the record exists before marking it complete.

---

## 10 Second SLA Q&A

**Q: POST /payment goes through API → Queue → Worker → Saga → Ledger → Payment Table. How do you guarantee completion within 10 seconds?**

Answer: Return 202 Accepted immediately after publishing to SQS. Client polls GET /payment/{id}/status or receives a webhook. The 10 second SLA means the payment reaches COMPLETED status within 10 seconds — not that the HTTP response takes 10 seconds.

Two notification approaches:
- Polling: client calls GET /payment/status every second
- Webhook: system calls client callback URL when complete

Scale Payment Workers horizontally based on SQS queue depth to keep processing time within budget.

---

## Vault Core Comparison

Traditional architecture: Payment Platform → Ledger System (two separate services)

Vault Core: Smart Contract → Vault Core Ledger (one system)

- Smart contracts written in Python define the payment rules
- Vault Core executes the smart contract and writes atomic postings to the immutable ledger
- Business logic lives in the smart contract, not in a payment worker or saga coordinator
- Streaming Ledger publishes events downstream

> "Vault Core removes the need for a separate payment orchestration layer. The smart contract defines the business rules and Vault Core guarantees atomic posting to the ledger."

---

## Key Gaps Identified This Session

1. Missed clarifying domestic vs international and payment SLA during requirements
2. Back of envelope maths error — 4.3M instead of 432M
3. Cache miss fallback must go to Read Replica, not Write Primary
4. Fraud / AML / Sanctions checks missing from diagram — mandatory in a real payment platform
5. Rail connectors missing — Faster Payments, CHAPS, BACS, SWIFT
6. Payment state machine should have named states: CREATED → VALIDATED → FUNDS_RESERVED → SUBMITTED → SETTLED
7. Fail closed on fraud provider unavailability — reject if provider is down, do not proceed
