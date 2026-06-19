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
