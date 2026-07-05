# Banking Ledger System — Interview Q&A

> Live mock interview session. Questions asked by interviewer, answers given and gaps filled.

---

## Requirements Gathering

**Q: Design a Banking Ledger System.**

Clarifying questions asked:
- How many transactions (TPS) does this system handle at peak?
- How many users are expected — registered and DAU?
- Is this an authenticated system only?
- Does consistency matter more than availability?

Answers given:
- Peak TPS: 10,000
- 50M registered users, 5M DAU
- Yes, auth handled by separate identity service — not in scope
- Correct — banking chooses CP over AP

---

## Functional Requirements

- Create a transaction (POST /payment)
- Read current balance (GET /balance)
- Audit trail — query all transactions for an account between two dates (added by interviewer)

---

## Non-Functional Requirements

- Consistency over availability — CP, not AP
- Idempotency — no duplicate payment processing
- Transactions must be queryable for compliance
- SLO target: 99.9% uptime

**Gap corrected:** CAP theorem — you cannot have consistency, availability, AND partition tolerance. Banking chooses CP. 99.9% uptime is an SLO target, separate from CAP.

---

## Back of Envelope Calculations

**Throughput:**
- Peak TPS: 10,000
- Average TPS: 1,000 (assume peak is not sustained all day)
- Daily volume at average: 1,000 * 86,400 = 86M transactions/day
- Daily volume at peak: 10,000 * 86,400 = 864M transactions/day

**User sanity check:**
- 5M DAU, 86M transactions/day → 86M / 5M = 17 transactions per user per day

**Storage:**
- Double-entry = 2 ledger records per transaction
- 1 record = 2KB
- 17 transactions * 2 entries * 2KB = 68KB per user per day
- 5M DAU * 68KB = 340GB/day
- 340GB * 365 = 124TB/year
- With 3 replicas: 378TB/year

**Write throughput:**
- 10,000 TPS * 2 entries * 2KB = 40MB/sec at peak
- Implication: single database cannot handle this — sharding required

**Gap corrected:** Original answer missed the double-entry factor — each transaction creates 2 records not 1.

---

## High Level Design — Q&A

**Q: Walk me through every step a single transaction takes from client to database.**

Answer:
- Request hits API Gateway — validated, auth checked, rate limited
- Idempotency key checked in Redis — if seen before, return original response
- If new, write debit and credit entries to PostgreSQL in a single ACID transaction
- Write outbox event in the same transaction
- Return 202 Accepted to client
- Outbox worker publishes event to SQS
- Payment worker polls SQS and processes downstream

**Q: Account A on shard 1, Account B on shard 2 — how do you guarantee both sides of double-entry are recorded?**

Answer: Saga pattern choreography — debit shard 1, on success publish event, credit shard 2.

**Gap corrected:** Conflict spotted between ACID transaction answer and Saga answer. Resolution:
- Same shard → single ACID transaction
- Different shards → Saga pattern with compensating transactions

**Q: 40MB/sec write load hitting PostgreSQL — what absorbs the peak?**

Answer: SQS message queue. API writes to queue, worker polls and processes, scales worker instances based on queue depth.

---

## Read Path — Q&A

**Q: 5M DAU checking balance — how do you serve it efficiently?**

Answer: GET /balance hits read replica.

**Gap corrected:** Too simple. Read replica with pre-computed balance table — no SUM() across billions of rows on every request. Balance updated every time a transaction is written (materialised view).

**Q: 5M users hitting GET /balance at 9am — read replica still hammered. What do you add?**

First answer: Rate limiting.

**Gap corrected:** Rate limiting controls abuse, not legitimate load. Answer is Redis cache — cache the balance with short TTL.

**Q: What TTL for balance cache and what is the trade-off?**

Answer: 60 seconds TTL.

Trade-off: User may see stale balance for up to 60 seconds after a transaction. Acceptable for reads. The write path is always strongly consistent — money never moves on stale data.

---

## Outbox Pattern — Q&A

**Q: Where does the outbox pattern fit and why do you need it alongside SQS?**

Answer: Did not know.

**Gap filled:** If SQS is down when the payment arrives, the message is lost. Outbox pattern writes to the outbox table in the same ACID transaction as the ledger entries. Outbox worker publishes to SQS when SQS is available. You can never have a payment without an outbox entry.

Rule: whenever a message queue is in a financial system design, the interviewer will ask "what if the queue is down?" Outbox pattern is always the answer.

---

## Key Gaps Identified This Session

1. Double-entry means 2 records per transaction — always factor this into storage calculations
2. CAP theorem — do not mix SLO uptime target with CAP choice
3. ACID vs Saga — must clarify same-shard vs cross-shard before answering
4. Balance reads — never SUM() on every request — pre-computed materialised view
5. Cache for reads — rate limiting is not the answer to read load
6. Outbox pattern — always needed when a queue sits next to a database in a financial system
