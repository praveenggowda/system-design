# System Design Pattern Library
# My System Design Bible

## Mindset

Just like I don't want to memorize 1000 LeetCode problems, I don't want to
memorize 100 system design case studies.

LeetCode:

    Question
        ↓
    Recognize Pattern
        ↓
    Apply Solution

System Design:

    Problem
        ↓
    Recognize Pattern
        ↓
    Apply Architecture

Don't memorize Uber, Netflix, WhatsApp, Twitter, Spotify.

Instead, master reusable patterns.

## Pattern 1: Request Flow

Problem: How does a request travel through the system?

    Client
        ↓
    Load Balancer
        ↓
    API Gateway
        ↓
    Service
        ↓
    Database

Learn: Load Balancer, API Gateway, Service Layer, Database.

## Pattern 2: Caching

Problem: Database becomes slow.

    Client
        ↓
    Cache
        ↓
    Database

Learn: Redis, CDN, In-Memory Cache.

## Pattern 3: Queue

Problem: A task takes several seconds. Don't make the user wait.

    API
        ↓
    Queue
        ↓
    Worker
        ↓
    Email / Notification

Learn: Amazon SQS, RabbitMQ, Background Workers.

## Pattern 4: Pub/Sub

Instead of:

    Payment Service
        ↓
    Notification, Analytics, Fraud Detection

Use:

    Payment Event
        ↓
    Event Bus
        ↓
    Subscribers

Learn: Kafka, SNS, EventBridge.

## Pattern 5: Database Scaling

When the database becomes too large.

Learn: Read Replicas, Sharding, Partitioning.

## Pattern 6: Stateless Services

Instead of storing user sessions inside the application server, store in:
- JWT
- Redis
- Database

Any server can now process the request.

## Pattern 7: Rate Limiting

Problem: 10,000 requests per second.

Solutions: Token Bucket, Sliding Window, Fixed Window.

## Pattern 8: Load Balancing

One server to ten servers to a hundred servers.

Learn: Round Robin, Least Connections, Weighted Load Balancing.

## Pattern 9: Reliability

Learn: Retry, Timeout, Circuit Breaker, Failover, Replication, High Availability.

## Pattern 10: Idempotency

The most important concept in financial systems.

    Payment Request
        ↓
    Unique Transaction ID
        ↓
    Already Processed?
        ↓
    Ignore Duplicate

## Pattern 11: Consistency

Question: Consistency or Availability?

Financial systems choose Consistency.

## Pattern 12: Event Sourcing

Instead of storing: Balance = £100

Store every event:

    Deposit £1000
    Withdraw £200
    Interest £10
    Fee £5

Replay all events to compute the current balance.

## Pattern 13: CQRS

Separate Writes from Reads.

    Writes → Commands
    Reads  → Queries

## Pattern 14: Distributed Lock

Prevent two users buying the last ticket. Only one succeeds.

## Pattern 15: Leader Election

Only one node performs: Cleanup, Backup, Scheduled Jobs, Cron Tasks.

## Pattern 16: Storage

Know when to use:

- SQL Database
- NoSQL Database
- File Storage
- Object Storage (S3)
- Block Storage

## Pattern 17: API Design

Learn: REST, Versioning, Pagination, Filtering, Error Handling, Idempotency.

## Pattern 18: Security

Learn: Authentication, Authorization, Encryption, Secrets Management, API Security.

## Pattern 19: Monitoring

Learn: Logging, Metrics, Tracing, Alerting.

## Pattern 20: Trade-offs

Every architecture decision has trade-offs.

Always answer:
- Why this design?
- What are the drawbacks?
- How does it scale?
- What changes if traffic grows 100x?

## Example 1: Design Uber

Don't think: Uber.

Think:

    Location
        ↓
    WebSocket
        ↓
    Pub/Sub
        ↓
    Matching Service
        ↓
    Queue
        ↓
    Database
        ↓
    Cache

It is simply a combination of patterns.

## Example 2: Design a Banking Ledger

Don't think: Ledger.

Think:

    API
        ↓
    Validation
        ↓
    Idempotency
        ↓
    Transaction
        ↓
    Database
        ↓
    Events
        ↓
    Read Model

Again, just a combination of patterns.

## Learn by Building Projects

### Project 1: Ledger Engine
Concepts: Domain Modelling, Validation, Business Rules, Idempotency, Testing.

### Project 2: Payment Processor
Concepts: Multi-Account Transactions, Atomicity, Consistency, Validation, Error Handling, Idempotency.

### Project 3: Rate Limiter
Concepts: Sliding Window, Token Bucket, Time-Based Algorithms, Performance.

### Project 4: HTTP Server
Concepts: Request Flow, Routing, Serialization, API Design, HTTP.

### Project 5: Event Queue
Concepts: Producer, Consumer, Retry, Dead Letter Queue, Pub/Sub.

### Project 6: LRU Cache
Concepts: HashMap, Linked List, Cache Eviction, Performance.

## Final Goal

Don't memorize companies. Don't memorize architectures. Recognize patterns.

Whenever you are asked to design a system, think:

What patterns does this problem need?

    [ ] Request Flow
    [ ] Caching
    [ ] Queue
    [ ] Pub/Sub
    [ ] Database
    [ ] Idempotency
    [ ] Rate Limiting
    [ ] Load Balancer
    [ ] Monitoring
    [ ] Security
    [ ] Trade-offs

Then combine those patterns to build the architecture.

## Long-Term Goal: Pattern Library

For every pattern, document:

- What problem does it solve?
- When should I use it?
- When should I NOT use it?
- Advantages
- Disadvantages
- Trade-offs
- Simple architecture diagram
- Mini Python implementation
- AWS services involved
- Real-world examples (Stripe, Wise, Uber, Netflix, Amazon)
- Common interview questions

Goal: Master 20 reusable patterns instead of memorizing 100 different system designs.
