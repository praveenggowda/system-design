# Booking System Design — Learnings

## What This Design Teaches

The booking system is one of the most important system designs to master because it covers:
- Concurrency and double booking prevention
- Distributed transactions
- Async messaging with guaranteed delivery
- Caching strategies
- Microservices decomposition

---

## Core Problem — Double Booking

Two users booking the last seat at the same millisecond.

**Pessimistic Locking** — lock the row before reading. Other users wait.
- Good for: low concurrency, financial systems, long transactions
- Bad for: high traffic — locked rows cause queues, timeouts, connection pool exhaustion

**Optimistic Locking** — no lock. Add a version column. Check version at write time.
```sql
UPDATE slots
SET available_seats = available_seats - 1, version = version + 1
WHERE slot_id = 1001 AND version = 42
```
If version changed, 0 rows updated — conflict detected. Show user "seat no longer available."
- Good for: high concurrency, short transactions, booking systems
- Bad for: high conflict rate — too many retries

**Rule:** Use optimistic locking for booking systems at scale.

---

## Microservices Decomposition

Do not put everything in one "Booking System" box. Break by responsibility:

| Service | Responsibility |
|---|---|
| Experience Service | Browse, search, list activities and availability |
| User Service | Registration, login, profile management |
| Booking Service | Create booking, check availability, optimistic lock |
| Payment Service | Process payment, handle refunds, talk to Stripe/PayPal |
| Notification Service | Send confirmation emails and reminders |

---

## Database Strategy

**Primary DB** — all writes go here. Bookings, user data, availability updates.

**Read Replica** — all reads go here. Browsing, searching, listing experiences. Async replication from Primary.

**Problem with Read Replica for availability:** replication lag means stale data. User B might see 1 seat when User A already booked it.

**Solution:** Cache availability in Redis with short TTL. Update Redis immediately on booking. Reads check Redis first.

---

## Caching Strategy — Cache Aside

```
Request comes in
→ Check Redis first
→ Cache hit → return immediately (no DB call)
→ Cache miss → read from Primary DB → store in Redis → return
```

When a booking is made → update Redis immediately with new seat count.

---

## Read vs Write Traffic

80% read (browsing), 20% write (bookings).

- Reads → Redis Cache → Read Replica
- Writes → PostgreSQL Primary

---

## Async Messaging — SQS

Booking Service and Notification Service should NOT call each other directly.

**Why:** if Notification Service is down, booking fails. Direct coupling is fragile.

**Solution:** Booking Service publishes to SQS. Notification Service polls SQS independently.

```
Booking Service → SQS → Notification Service → SES → Email
```

Decoupled. If Notification Service is down, messages wait in SQS until it recovers.

---

## Guaranteed Delivery — Outbox Pattern

**Problem:** Booking Service writes to DB successfully. Then crashes before publishing to SQS. User never gets confirmation email.

**Solution — Outbox Pattern:**

```
In ONE database transaction:
1. Write booking to bookings table
2. Write event to outbox table (status = pending)
Both succeed or both rollback — atomic.

Separately:
3. Outbox Worker polls outbox table every few seconds
4. Publishes pending events to SQS
5. Marks event as published = true
```

If service crashes after step 2, outbox row is still there. Worker picks it up on restart. Nothing lost.

**You built this at Stepstone** — this is your strongest story in any interview.

---

## Dead Letter Queue (DLQ)

Separate from Outbox Pattern. DLQ handles messages that are already in SQS but fail to process after retries.

```
SQS → Notification Service fails 3 times → message moves to DLQ
```

DLQ holds failed messages for investigation and reprocessing. Prevents poison pill messages from blocking the queue forever.

---

## CDN

Static content (images, JS, CSS) served from CDN edge locations close to the user.

```
User → CDN (nearest edge) → serves static content
User → CDN → API Gateway → services (only for dynamic requests)
```

Reduces latency and API Gateway load.

---

## Full Architecture Summary

```
User
 ↓ HTTP Request
CDN (static content)
 ↓ Forward Request
API Gateway (auth, rate limiting, routing)
 ↓
┌─────────────────────────────────────────────┐
│  Experience    User      Booking    Payment  │
│  Service       Service   Service   Service  │
│                           ↓          ↓      │
│                        Outbox      PayPal   │
│                        Table               │
└─────────────────────────────────────────────┘
        ↓              ↓
   Redis Cache    PostgreSQL Primary
        ↓              ↓
   Read Replica    Outbox Worker
                       ↓
                      SQS ←→ DLQ
                       ↓
              Notification Service
                       ↓
                      SES
                       ↓
                     Email
```

---

## Topics Covered in This Design

| Topic | Concept |
|---|---|
| Concurrency | Optimistic vs pessimistic locking |
| Database | Primary, Read Replica, replication lag |
| Caching | Cache aside, Redis, TTL, stale data |
| Microservices | Decomposition by responsibility |
| Async messaging | SQS, decoupling, poll vs push |
| Guaranteed delivery | Outbox Pattern, atomic transaction |
| Failure handling | DLQ, retry, poison pill |
| Performance | CDN, Read Replica, Redis |
| Payment | Third party integration (PayPal/Stripe) |
| Notifications | SES, async email delivery |

---

## Key Interview Phrases

- "I would use optimistic locking here because at this scale pessimistic locking causes connection pool exhaustion under peak traffic"
- "The Outbox Pattern guarantees exactly-once delivery by writing the event atomically with the booking in the same database transaction"
- "We separate reads and writes — reads go to Redis cache then Read Replica, writes always go to Primary"
- "SQS decouples Booking Service from Notification Service — if Notification is down, messages wait in the queue and nothing is lost"
- "DLQ captures messages that fail after retries so we can investigate without losing data"
