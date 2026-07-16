# Notification System Design — Learnings

## What This Design Teaches

The notification system is one of the most important designs to master because it covers:
- Event-driven architecture
- Async messaging at scale
- Idempotency and deduplication
- Retry and circuit breaker patterns
- Outbox Pattern for guaranteed delivery
- Multi-channel delivery (email, SMS, push)

**You built this at Stepstone — 10 million notifications per day across 16 European markets. This is your strongest story.**

---

## Core Problem

A user action (job application, booking confirmation, password reset) must trigger a notification reliably. The notification must be delivered exactly once even if the system crashes, retries, or has network failures.

---

## Architecture Overview

```
Trigger (HTTP event / Kafka / S3 / Hangfire batch)
 ↓
API / Event Consumer
 ↓
Outbox Table (PostgreSQL) — atomic write
 ↓
Outbox Worker — polls pending events
 ↓
SQS
 ↓
Notification Consumer — circuit breaker, retry
 ↓
SES / SNS / Push provider
 ↓
User (email / SMS / push)
```

---

## Four Trigger Patterns

| Pattern | Trigger | Use Case |
|---|---|---|
| HTTP | REST API call | Password reset, user action |
| Kafka | Event stream | Real-time job match, booking confirmed |
| S3 | File upload | Bulk import, CSV processing |
| Hangfire batch | Scheduled job | Daily digest, reminders |

---

## Outbox Pattern — Guaranteed Delivery

**Problem:** Service writes to DB but crashes before publishing to SQS. Message lost.

**Solution:**
```
In ONE database transaction:
1. Write notification intent to outbox table (status = pending)
2. Both succeed or both rollback — atomic

Separately:
3. Outbox Worker polls outbox table every few seconds
4. Publishes pending events to SQS
5. Marks as published = true
```

If service crashes between steps 1 and 4, the outbox row survives. Worker picks it up on restart.

**This guarantees at-least-once delivery.**

---

## Idempotency — Prevent Duplicate Notifications

At-least-once delivery means the same message could be processed twice. You must deduplicate.

**Solution — Redis idempotency key:**
```
Key: notification:{userId}:{eventType}:{eventId}
TTL: 24 hours

Before processing:
→ Check Redis for key
→ Key exists → already processed → skip
→ Key missing → process → set key in Redis
```

Same message delivered twice — second delivery hits Redis key, skipped. User gets exactly one notification.

---

## Circuit Breaker — Polly

If SES or downstream provider is down, do not keep hammering it. Apply circuit breaker.

```
States:
Closed   → requests flow normally
Open     → requests fail immediately (fast fail)
Half-Open → allow one test request through

Closed → too many failures → Open
Open → wait timeout → Half-Open
Half-Open → success → Closed
Half-Open → failure → Open
```

**Why it matters:** Without circuit breaker, a slow SES causes your service threads to pile up waiting. Connection pool exhausts. Service goes down. Circuit breaker fails fast and protects your service.

---

## Retry with Exponential Backoff

Transient failures (network blip, SES rate limit) should be retried. But do not retry immediately.

```
Attempt 1: fail → wait 1 second
Attempt 2: fail → wait 2 seconds
Attempt 3: fail → wait 4 seconds
Attempt 4: fail → move to DLQ
```

Exponential backoff prevents hammering a struggling downstream service.

---

## Dead Letter Queue (DLQ)

Messages that fail after all retries move to DLQ. They are not lost — they wait for investigation.

```
SQS → Consumer fails 3 times → message moves to DLQ
```

Ops team investigates DLQ. Fix the issue. Replay messages.

---

## Multi-Channel Delivery

| Channel | Provider | Use Case |
|---|---|---|
| Email | AWS SES | Booking confirmations, receipts |
| SMS | SNS / Twilio | OTP, urgent alerts |
| Push | Firebase FCM | Mobile app notifications |

Route by notification type and user preference. User may opt out of email but keep push.

---

## Scale Considerations

**10 million notifications per day = ~116 per second average. Peak could be 10x.**

- SQS scales automatically — no capacity to manage
- Multiple Consumer instances in ECS — scale by CPU or queue depth
- SES sending limits — request quota increase, use dedicated IPs for high volume
- Partition Kafka by userId — ensures ordering per user, avoids hot partitions

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Sync vs Async | Async via SQS | Decouple trigger from delivery, handle failures independently |
| At-least-once vs exactly-once | At-least-once + idempotency | Exactly-once is hard, cheaper to deduplicate at consumer |
| Push vs pull | SQS pull | Consumer controls pace, no overload |
| Retry strategy | Exponential backoff | Avoids hammering struggling downstream |
| Failure handling | DLQ | No data loss, investigate and replay |

---

## Topics Covered in This Design

| Topic | Concept |
|---|---|
| Event-driven architecture | Triggers, consumers, decoupling |
| Outbox Pattern | Atomic write, guaranteed delivery |
| Idempotency | Redis deduplication, exactly-once semantics |
| Circuit breaker | Polly, Closed/Open/Half-Open states |
| Retry | Exponential backoff, transient failures |
| Dead Letter Queue | Failure handling, no data loss |
| Multi-channel | SES, SNS, FCM routing |
| Scaling | SQS auto-scale, ECS horizontal scaling |

---

## Key Interview Phrases

- "The Outbox Pattern guarantees at-least-once delivery by writing the event atomically with the business data in the same transaction"
- "We use a Redis idempotency key to convert at-least-once into exactly-once at the consumer level"
- "Circuit breaker protects our service from cascading failures when SES is degraded — we fail fast instead of exhausting our thread pool"
- "DLQ ensures no message is ever permanently lost — failed messages wait for investigation and replay"
- "We partition Kafka by userId to maintain ordering per user and avoid hot partitions on popular event types"
# Notification System Design

Source: Alex Xu, System Design Interview Vol 1, Chapter 10.

## The Core Architecture

```
API Servers
    |
Notification Service
    |
Message Queue (one per channel)
    |
Workers
    |
Third-party providers
    |
Devices / Inboxes
```

## Supported Channels

- Push notifications: APNs (Apple Push Notification Service) for iOS, FCM (Firebase Cloud Messaging) for Android
- SMS: Twilio, Nexmo, or similar providers
- Email: SendGrid, Mailchimp, or similar providers

Each channel has its own queue and its own set of workers. They scale independently.

## Key Components

### Notification Service

Receives notification requests from internal services or API servers.
Validates the request, fetches user data, applies the template, and routes to the correct queue.

### Message Queues

One queue per channel (push, SMS, email).
Decouples the notification service from the delivery workers.
Workers consume at their own pace. If a provider is slow or down, the queue absorbs the spike.

### Workers

Pull messages from the queue and call the third-party provider.
Handle retries on failure.
Log the result to the notification log.

### User Info and Device Token Store

Workers need to know where to send the notification.
User preferences (opted out of email? language?), device tokens for push, and phone numbers for SMS are fetched here.

### Notification Templates

Pre-defined message structures with placeholders.
Ensures consistency across channels and reduces the risk of bad content being sent.

### Notification Log

Every notification attempt is recorded: channel, status, timestamp, provider response.
Used for debugging, auditing, and retry logic.
Without this, you cannot answer "did this user receive this notification?"

## Scaling Considerations

Single notification server is a single point of failure. Move to multiple servers behind a load balancer.

Message queues are the key to scaling. Workers can scale horizontally based on queue depth.

Database reads are expensive at scale. Cache user preferences and device tokens.

Third-party providers have rate limits. Multiple providers per channel give you fallback and higher throughput.

## Failure Handling

Workers must be idempotent. If a message is processed twice the user should not receive the notification twice.

Use a deduplication ID (notification ID) to detect and reject duplicates.

Dead letter queues for messages that fail after max retries. Investigate separately rather than losing them.

## Rate Limiting

Do not send too many notifications to the same user in a short window.
Users opt out when they feel spammed.
Implement per-user rate limits per channel (e.g. max 3 push notifications per hour).
Soft limits warn, hard limits drop or delay the message.

## Retry Mechanism

Workers retry on failure with exponential backoff.
Max retry attempts before moving to dead letter queue.
Retries must be idempotent — the same notification cannot be delivered twice because a retry fired.
Track attempt count and last error in the notification log.

## Security

Only authenticated internal services can call the notification service.
Use app keys or internal tokens — never expose the notification API publicly.
Validate all inputs: recipient exists, template exists, content is within limits.
Do not log sensitive content (PII, financial data) in notification logs.

## Monitoring

Track at every stage:
- Messages enqueued per channel
- Worker throughput (messages processed per second)
- Provider response times
- Delivery success rate per channel
- Bounce rate and failure rate

Alert on: queue depth growing (workers falling behind), delivery rate dropping, provider error rate spiking.

## Event Tracking

Track the full lifecycle of each notification:
- Created
- Queued
- Sent to provider
- Delivered (provider confirmed)
- Opened (if applicable — email open tracking, push tap)
- Failed

This data feeds analytics, debugging, and SLA reporting.

## Notification Settings

Users control their preferences:
- Opt in or out per channel (email yes, SMS no)
- Opt in or out per notification type (marketing no, transactional yes)
- Delivery time preferences (not between 10pm and 8am)

Check preferences before sending. Respect them. Sending a notification a user opted out of is a compliance issue, not just a product issue.

## Notification Templates

Templates separate content from code.
A template has: subject, body with placeholders, channel, language.
Workers resolve placeholders at send time using user data.
Templates are versioned. A bad template change should not require a code deployment to fix.

## Exactly Once Delivery

This is hard. Networks are unreliable. Providers do not always confirm.

The practical goal is at-least-once delivery with deduplication on the consumer side.

How:
- Each notification has a unique ID
- Before sending, check if this ID was already delivered successfully (check the log)
- If yes, skip it
- If no, send and record the result

At-least-once plus deduplication gives you effectively-once in most cases.
Exactly-once in a distributed system is theoretically impossible without coordination overhead that is not worth it for notifications.

## Reliability

Single points of failure to eliminate:
- Single notification server: add more, put behind load balancer
- Single queue: managed queue service (SQS, Kafka) with replication
- Single worker: run multiple instances
- Single provider: have fallback providers per channel

Data persistence: notification log must be durable. If a worker crashes mid-send, you need to know whether the notification was delivered or not.

Graceful degradation: if email provider is down, queue the messages and retry. Do not drop them.

## Key Takeaway

The message queue is the most important architectural decision in this system.

Without it: one slow provider blocks everything, one spike brings down the service.
With it: channels are isolated, workers scale independently, failures are contained.

Every component downstream of the queue is stateless and replaceable.
The queue is the buffer between what you promise (deliver this notification) and what the world gives you (flaky providers, rate limits, network failures).

