# System Design

System design diagrams and interview practice covering distributed systems, scalability and real-world architecture patterns.

Each folder contains a `PROBLEM.md` (the design prompt), a draw.io diagram, and an `interview-qa.md` where available (mock interview Q&A with gaps identified).

## Interview Practice

Folders with full mock interview Q&A:

| System | Key Concepts |
|---|---|
| [Banking Ledger](./banking-ledger/) | Double-entry bookkeeping, outbox pattern, sharding, pre-computed balance, CQRS |
| [Payment System](./payment-system/) | Payment state machine, Saga pattern, idempotency, rail connectors, 202 Accepted |
| [Event Processor](./event-processor/) | Outbox pattern, SNS fan-out to SQS, at-least-once delivery, idempotent consumers, DLQ |
| [Real-Time Balance](./real-time-balance/) | Redis pub/sub cache invalidation, Write Primary reads, pre-computed balance table, CP |

## Study Designs

Folders from Alex Xu System Design Interview (Vol 1 & 2):

| System | Key Concepts |
|---|---|
| [Notification System](./notification-system/) | Kafka, SQS, Outbox Pattern, Circuit Breaker, DLQ, Push/Email/SMS |
| [Rate Limiter](./rate-limiter/) | Fixed Window, Sliding Window, Redis INCR + TTL, fail open vs fail closed |
| [URL Shortener](./url-shortener/) | DynamoDB, Redis LRU cache, Base62 encoding, redirect flow |
| [Chat System](./chat-system/) | WebSocket, Redis Pub/Sub, multi-server routing, PostgreSQL, Push Notifications |
| [News Feed](./news-feed/) | Fan-out on write vs read, Redis cache, pagination |
| [Booking System](./booking-system/) | Distributed locking, idempotency, seat reservation, race conditions |

## Reference

| Resource | Topics |
|---|---|
| [Banking Concepts Q&A](./interview-prep/banking-concepts-qa.md) | Double-entry, idempotency, Saga, event sourcing, CQRS, outbox, CAP, API design, observability |
| [Interview Framework](./interview-prep/system-design-interview-framework.md) | Clarify, back of envelope, high level, deep dive, failure handling |
| [Pattern Library](./interview-prep/system-design-pattern-library.md) | Common patterns with when to use and trade-offs |

## Tools

Diagrams created with [draw.io](https://app.diagrams.net).

## About

14 years of backend engineering experience building distributed systems at scale. These designs reflect real production patterns used in systems processing millions of events per day.
