# System Design

System design diagrams and learnings covering distributed systems, scalability and real world architecture patterns.

Each folder contains a draw.io diagram and a learnings document covering design decisions, trade-offs and key components.

## Designs

| System | Topics Covered |
|---|---|
| [Notification System](./notification-system/) | Kafka, SQS, Outbox Pattern, Circuit Breaker, DLQ, Push/Email/SMS |
| [Rate Limiter](./rate-limiter/) | Fixed Window, Redis INCR + TTL, API Gateway, fail open vs fail closed |
| [URL Shortener](./url-shortener/) | DynamoDB, Redis LRU cache, Base62 encoding, redirect flow |
| [Chat System](./chat-system/) | WebSocket, Redis Pub/Sub, multi-server routing, PostgreSQL, Push Notifications |
| [News Feed](./news-feed/) | Fan-out on write vs read, Redis cache, pagination |
| [Booking System](./booking-system/) | Distributed locking, idempotency, seat reservation, race conditions |

## Tools

Diagrams created with [draw.io](https://app.diagrams.net).

## About

14 years of backend engineering experience building distributed systems at scale. These designs reflect real production patterns used in systems processing millions of events per day.
