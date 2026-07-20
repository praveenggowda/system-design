# System Design

A collection of distributed system designs covering real-world architecture patterns, scalability, reliability and consistency.

Each folder contains a problem statement, architecture notes, and a diagram.

## Designs

| System | Key Concepts |
|---|---|
| [Banking Ledger](./banking-ledger/) | Double-entry bookkeeping, outbox pattern, sharding, pre-computed balance, CQRS |
| [Payment System](./payment-system/) | Payment state machine, Saga pattern, idempotency, rail connectors, 202 Accepted |
| [Event Processor](./event-processor/) | Outbox pattern, SNS fan-out to SQS, at-least-once delivery, idempotent consumers, DLQ |
| [Real-Time Balance](./real-time-balance/) | Redis pub/sub cache invalidation, Write Primary reads, pre-computed balance table, CP |
| [Metrics and Logging](./metrics-and-logging/) | Kafka, Elasticsearch, InfluxDB, Grafana, Kibana, S3 cold storage, backpressure, tiered retention |
| [Webhook Delivery](./webhook-delivery/) | SQS, DynamoDB registry, PostgreSQL status tracking, exponential backoff, at-least-once delivery, SELECT FOR UPDATE |
| [Notification System](./notification-system/) | SQS, SNS fan-out, Outbox Pattern, Redis idempotency, Priority queues, DLQ, Push/Email/SMS — two variants |
| [Automation System](./automation-system/) | Kafka, DynamoDB rule store, Redis cache, idempotency, SNS fan-out, SQS per service, time-based scheduler, DLQ, audit trail |
| [Item Reordering](./item-reordering/) | Gap-based integers, optimistic locking, version number, Redis Pub/Sub, WebSockets, real-time fan-out, conflict resolution, optimistic UI |
| [Rate Limiter](./rate-limiter/) | Fixed Window, Sliding Window, Redis INCR + TTL, fail open vs fail closed |
| [URL Shortener](./url-shortener/) | DynamoDB, Redis LRU cache, Base62 encoding, redirect flow |
| [Chat System](./chat-system/) | WebSocket, Redis Pub/Sub, multi-server routing, PostgreSQL, Push Notifications |
| [News Feed](./news-feed/) | Fan-out on write vs read, Redis cache, pagination |
| [Real-Time Board](./real-time-board/) | WebSockets, Redis Pub/Sub, optimistic locking, Outbox, write-through cache, per-item cache invalidation |
| [Transaction Processing](./transaction-processing/) | Double-entry bookkeeping, idempotency, optimistic locking, Outbox, Saga for cross-shard transfers, two ingestion paths |
| [Search System](./search-system/) | CDC, Kafka, Elasticsearch, document-level permissions, idempotent indexing, query-level cache |
| [Booking System](./booking-system/) | Distributed locking, idempotency, seat reservation, race conditions |

## Reference

| Guide | Topics |
|---|---|
| [System Design Patterns](./interview-prep/system-design-pattern-library.md) | Common patterns with when to use and trade-offs |
| [Banking and Fintech Concepts](./interview-prep/banking-concepts-qa.md) | Double-entry, idempotency, Saga, event sourcing, CQRS, outbox, CAP, API design, observability |

## All Topics

A full index of every concept covered — storage, messaging, consistency, caching, API design, observability, and more: [TOPICS.md](./TOPICS.md)

## Tools

Diagrams created with [draw.io](https://app.diagrams.net) and [Excalidraw](https://excalidraw.com).

## About

14 years of backend engineering experience building distributed systems at scale. These designs reflect real production patterns used in systems processing millions of events per day.
