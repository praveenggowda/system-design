# Topics Covered

A full index of every concept covered across all system designs in this repo.

---

## Storage

| Topic | Where |
|---|---|
| PostgreSQL — sharding, read replicas, write primary | Banking Ledger, Real-Time Balance |
| Pre-computed balance table | Banking Ledger, Real-Time Balance |
| Outbox table — atomic write, PENDING/PROCESSING/PROCESSED states | Banking Ledger, Event Processor |
| Elasticsearch — full-text search, hot/cold tiering | Metrics and Logging |
| InfluxDB — time-series, downsampling | Metrics and Logging |
| DynamoDB — partition key, single-table design | URL Shortener |
| Redis — cache, pub/sub, sorted sets, TTL | Real-Time Balance, Chat System, News Feed, Rate Limiter |
| S3 — cold archive storage, Athena for ad-hoc queries | Metrics and Logging |

---

## Messaging and Queues

| Topic | Where |
|---|---|
| Kafka — high throughput, multiple consumers, topics | Metrics and Logging, Notification System |
| SQS — at-least-once delivery, DLQ | Event Processor, Payment System |
| SNS fan-out to SQS | Event Processor |
| Outbox pattern — write to DB then publish | Banking Ledger, Event Processor |
| Dead Letter Queue (DLQ) | Event Processor, Notification System |

---

## Consistency and Reliability

| Topic | Where |
|---|---|
| CAP theorem — CP vs AP | Banking Ledger, Real-Time Balance, Event Processor |
| Idempotency — event_id unique constraint, Redis check | Event Processor, Payment System, Real-Time Balance |
| At-least-once delivery | Event Processor |
| Idempotent consumers — Redis + DB check | Event Processor |
| Saga pattern — compensating transactions | Banking Ledger, Payment System |
| Atomic write — database transaction | Banking Ledger, Event Processor, Real-Time Balance |
| ACID — same shard | Banking Ledger |

---

## Caching

| Topic | Where |
|---|---|
| Redis cache hit/miss fallback pattern | Real-Time Balance, URL Shortener |
| Cache invalidation via pub/sub | Real-Time Balance |
| Cache TTL as safety net | Real-Time Balance |
| Read from Write Primary vs Read Replica | Real-Time Balance, Banking Ledger |
| LRU cache | URL Shortener |

---

## API Design

| Topic | Where |
|---|---|
| 202 Accepted — async response pattern | Event Processor, Payment System |
| Idempotency key in request header | Payment System |
| REST versioning — /v1/ | Payment System |
| Rate limiting at API Gateway | Metrics and Logging, Rate Limiter |
| Load Balancer → API Gateway pattern | All designs |

---

## Payment and Finance

| Topic | Where |
|---|---|
| Double-entry bookkeeping | Banking Ledger |
| Payment state machine — CREATED → SETTLED | Payment System |
| Rail connectors — SWIFT, SEPA, FPS | Payment System |
| Fraud and AML checks | Payment System |
| Pre-authorisation and funds reservation | Payment System |
| CQRS — separate read and write models | Banking Ledger |
| Event sourcing | Banking Ledger |

---

## Observability

| Topic | Where |
|---|---|
| Metrics — InfluxDB, Grafana dashboards | Metrics and Logging |
| Logs — Elasticsearch, Kibana | Metrics and Logging |
| Alerting — Grafana → PagerDuty/Slack | Metrics and Logging |
| Log tiering — hot (7 days) → cold S3 (30 days) | Metrics and Logging |
| Fluent Bit agent — batching at source | Metrics and Logging |
| P1 investigation order — metrics → traces → logs | Banking Concepts Q&A |

---

## Scalability

| Topic | Where |
|---|---|
| Horizontal sharding | Banking Ledger |
| Back of envelope — RPS, storage, throughput | All designs |
| Backpressure via queue buffering | Metrics and Logging |
| Fan-out on write vs read | News Feed |
| Pagination | News Feed |

---

## Real-Time Systems

| Topic | Where |
|---|---|
| WebSocket — persistent connection | Chat System |
| Redis pub/sub — real-time cache invalidation | Real-Time Balance, Chat System |
| Multi-server WebSocket routing | Chat System |

---

## Concurrency and Locking

| Topic | Where |
|---|---|
| Distributed locking | Booking System |
| Race condition handling | Booking System |
| Seat reservation with optimistic locking | Booking System |
