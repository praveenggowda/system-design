# Rate Limiter System Design — Learnings

## What This Design Teaches

The rate limiter introduces:
- Middleware placement in the API Gateway layer
- Redis INCR for atomic counter management
- TTL-based fixed window algorithm
- Response headers for client communication
- Fail open vs fail closed strategy

---

## Clarifying Questions to Ask

- Client side or server side rate limiting?
- How many requests allowed per window? (e.g. 100 requests per minute)
- Per user, per IP, or per API key?
- Single server or distributed environment?
- What to do on breach — 429, drop, or queue?
- Which endpoints to rate limit?

---

## Algorithms

| Algorithm | How it works | Trade-off |
|---|---|---|
| Fixed Window | Counter resets at fixed interval (0-60s, 60-120s) | Simple, slight spike at window boundary |
| Sliding Window | Rolling window from NOW | More accurate, more memory |
| Token Bucket | Bucket fills at fixed rate, each request consumes a token | Allows bursts |
| Leaky Bucket | Requests queued, processed at fixed rate | Smooths traffic |

**For interviews — use Fixed Window with Redis. Simplest to implement and defend.**

---

## Where Rules Live

Rules are NOT stored in Redis. Redis only stores the live counter.

Rules live in the Rate Limiter service config or a Rules Database:
```
user_tier: free    → 100 req/min
user_tier: premium → 1000 req/min
endpoint: /login   → 10 req/min
endpoint: /health  → unlimited
```

Rate Limiter loads rules at startup, caches in memory, refreshes periodically.

---

## Architecture

```
Client
  ↓ request
┌─────────────────────────────┐
│      API Gateway Layer      │
│  ┌──────────────────────┐   │
│  │     Rate Limiter     │ ←→ Redis (counter + TTL)
│  └──────────────────────┘   │
└─────────────────────────────┘
  ↓ allowed              ↓ throttled
Backend Servers        429 → Client
                       option 1 → Drop Request
                       option 2 → Message Queue
```

Rate Limiter sits inside the API Gateway layer as middleware. Requests are checked before reaching any backend service.

---

## Redis Storage

```
Key:   rate_limit:{userId}
Value: request count (integer) — incremented with INCR
TTL:   60 seconds (auto resets the window)
```

On each request:
1. `INCR rate_limit:userId` — atomic increment
2. If count == 1 → first request in window → `EXPIRE rate_limit:userId 60`
3. If count > 100 → reject with 429
4. If count <= 100 → allow through

INCR is atomic — safe under concurrent requests, no race conditions.

---

## Request Flow

```
1. User sends request to API Gateway
2. Rate Limiter intercepts before routing
3. Rate Limiter calls Redis: INCR rate_limit:userId
4. Redis returns new count
5. If count <= 100 → allow → forward to Backend Server
6. If count > 100 → throttle:
   - Return 429 with response headers
   - OR drop request silently
   - OR push to Message Queue for later processing
```

---

## Response Headers on 429

Always include these so the client knows when to retry:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1718200860   (unix timestamp when window resets)
```

---

## When Redis is Down

**Fail open** — allow the request through. Availability over strict accuracy. Correct for most APIs.

**Fail closed** — block the request. Safety over availability. Correct for financial APIs or sensitive endpoints.

> "If Redis is unavailable we fail open — allow requests through. For most APIs availability is more important than strict rate limiting accuracy. For financial APIs we would fail closed."

---

## Key Interview Phrases

- "The Rate Limiter sits as middleware inside the API Gateway layer — every request passes through it before reaching any backend service."
- "Redis stores the counter per user with a 60 second TTL. The INCR command is atomic so concurrent requests are handled safely — no race conditions."
- "Rules are stored in the Rate Limiter config, not in Redis. Redis only holds the live counter. This keeps Redis fast and focused."
- "On the 101st request we return 429 with X-RateLimit headers so the client knows when the window resets and can retry intelligently."
- "If Redis goes down we fail open — allow requests through. The alternative is taking down the entire API because the rate limiter is unavailable, which is worse."

---

## Topics Covered

| Topic | Concept |
|---|---|
| Algorithm | Fixed Window — counter + TTL in Redis |
| Placement | Middleware inside API Gateway layer |
| Counter storage | Redis INCR — atomic, O(1) |
| Window reset | TTL on Redis key — auto expires after 60s |
| Response | 429 with X-RateLimit headers |
| Throttle options | Drop, 429, or Message Queue |
| Failure mode | Fail open (most APIs) vs fail closed (financial) |
| Rules storage | Rate Limiter config/DB — not Redis |
# Rate Limiter

Source: Alex Xu, System Design Interview Vol 1, Chapter 4.

## Why Rate Limiting

- Prevent Denial of Service (DoS) attacks
- Reduce costs (fewer requests to third-party APIs means lower bills)
- Prevent servers from being overloaded

## Where to Put the Rate Limiter

Typically placed in the API Gateway layer.
The gateway sits between clients and backend services.
Every request passes through it before reaching any backend.

Client-side rate limiting is not reliable — clients can be modified.
Server-side is the correct approach.

## Algorithms

### Token Bucket

A bucket holds tokens up to a maximum capacity.
Tokens are added at a fixed rate.
Each request consumes one token.
If the bucket is empty the request is rejected.
Allows bursts up to the bucket size.

### Leaking Bucket

Requests enter a queue (the bucket).
Requests are processed at a fixed outflow rate.
If the queue is full incoming requests are dropped.
Smooths traffic. No bursts.

### Fixed Window Counter

Time is divided into fixed windows (e.g. 0-60s, 60-120s).
A counter tracks requests per window.
Counter resets at the start of each new window.
Simple to implement. Weakness: spike at window boundary.

### Sliding Window Log

Keeps a log of request timestamps.
On each request, remove timestamps older than the window.
Count remaining timestamps. If over the limit, reject.
More accurate than fixed window. Higher memory usage.

### Sliding Window Counter

Hybrid of fixed window and sliding window log.
Uses a weighted count from the previous window and current window.
More accurate than fixed window. Less memory than sliding window log.

## Single Server vs Distributed

On a single server rate limiting is straightforward.
One Redis instance, one counter per user.

In a distributed environment with multiple servers and multiple rate limiter instances, two problems arise:

Race condition: two servers read the same counter simultaneously, both allow the request, counter increments twice.
Solved with atomic operations (Redis INCR) or Lua scripts.

Synchronisation: each rate limiter has its own state.
If user requests hit different servers they may bypass the limit.
Solved by using a centralised data store (Redis) that all rate limiter instances read and write to.

## Performance Optimisation

Cache rules in memory locally on each rate limiter.
Reduce round trips to the rules database.
Refresh the cache periodically.

## Multi Data Centre

Route users to the nearest data centre.
Each data centre has its own Redis cluster.
Use eventual consistency to synchronise counters across data centres.

Eventual consistency means the total count across all data centres may temporarily exceed the limit during synchronisation delay.
This is an acceptable trade-off for most use cases.

## Synchronisation with Eventual Consistency

When running rate limiters across multiple data centres, counters are synchronised using eventual consistency.
The data will converge to the correct state but may be briefly out of sync.

## Monitoring

Track:
- How often the rate limit is being hit (by endpoint, by user, by tier)
- Whether the chosen algorithm is appropriate (too many false rejections?)
- Whether the rules are correctly configured (limits too tight or too loose?)

Monitoring tells you whether to adjust limits or change algorithms.

## Hard and Soft Rate Limiting

Hard rate limiting: requests exceeding the threshold are always rejected. No exceptions.

Soft rate limiting: requests can exceed the threshold by a small margin for a short period.
Useful for handling burst traffic without immediately rejecting legitimate users.
