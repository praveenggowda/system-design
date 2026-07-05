# URL Shortener Design — Learnings

## What This Design Teaches

The URL shortener is the best starter system design because it covers:
- Hashing and encoding
- Caching strategy
- Database design
- Read vs write scaling
- CDN and redirects
- Rate limiting

---

## Core Problem

Convert a long URL into a short 6-8 character code. When a user visits the short URL, redirect them to the original long URL.

```
https://tripadvisor.com/experiences/paris/boat-tour-seine-river-2024
→ https://trip.ad/x7Kp2q
```

---

## Functional Requirements

- Shorten a long URL → return short URL
- Redirect short URL → return original URL
- Short URL must be unique
- Short URL should be 6-8 characters

---

## Non-Functional Requirements

- 100 million URLs created per day (write)
- 10 billion redirects per day (read) — read heavy 100:1 ratio
- Low latency redirects — under 10ms
- 99.99% availability
- URLs should not expire unless specified

---

## The Hashing Problem

You need to convert a long URL to a 6 character code.

**Option 1 — MD5 Hash**
Hash the URL, take first 6 characters.
Problem: collisions. Two different URLs could produce the same 6 characters.

**Option 2 — Base62 Encoding**
Generate a unique ID (auto-increment or distributed counter). Encode it in Base62.

```
Base62 characters: a-z A-Z 0-9 = 62 characters
6 characters = 62^6 = 56 billion unique codes
```

Example: ID 12345 → base62 → `3d7K`

This is the correct approach — unique ID guarantees no collision.

---

## Database Design

**URLs Table:**
```
id          BIGINT PRIMARY KEY (auto-increment)
short_code  VARCHAR(8) UNIQUE
long_url    TEXT
created_at  TIMESTAMP
expires_at  TIMESTAMP (nullable)
user_id     BIGINT (nullable)
click_count BIGINT DEFAULT 0
```

**Index on short_code** — every redirect lookup uses this column. Must be O(1).

---

## Read vs Write Ratio

100:1 read to write. 10 billion redirects per day.

**Write path (create short URL):**
```
Client → API Gateway → URL Service → PostgreSQL Primary
```

**Read path (redirect):**
```
Client → CDN → Cache hit → 301 redirect (no server needed)
Client → CDN → Cache miss → API Gateway → Redis → PostgreSQL Read Replica
```

---

## Caching Strategy

Redis stores `short_code → long_url` mapping.

```
Redirect request comes in:
→ Check Redis for short_code
→ Cache hit → return long URL immediately (sub-millisecond)
→ Cache miss → read from Read Replica → store in Redis → return
```

**TTL:** 24 hours for popular URLs, shorter for rarely accessed ones.

**Cache hit rate target:** 99%+ — most popular URLs are accessed repeatedly.

---

## CDN for Redirects

For the most popular short URLs, cache the redirect at CDN edge nodes.

```
User in London visits trip.ad/x7Kp2q
→ CDN edge in London has the mapping cached
→ 301 redirect served from CDN (no request reaches origin server)
→ Under 5ms
```

---

## Redirect Types

| Type | Code | Caching | Use Case |
|---|---|---|---|
| Permanent | 301 | Browser and CDN cache | URL will never change |
| Temporary | 302 | Not cached | URL may change, need accurate click tracking |

Use 302 if you need accurate click count analytics — 301 means the browser caches and never calls your server again.

---

## Rate Limiting

Without rate limiting, one user could create millions of short URLs.

```
API Gateway rate limiter:
→ 100 URLs created per user per hour
→ Return 429 if exceeded
```

Redis counter with TTL — same pattern as the rate limiter system design.

---

## Handling Collisions (Base62)

With distributed systems, multiple servers generating IDs could produce duplicates.

**Solution — Centralized ID Generator:**
- Single counter in Redis or a dedicated ID service
- Each URL creation gets next increment
- Encode that ID in Base62

**Alternative — UUID + Base62:**
Generate UUID, convert to Base62. Lower collision risk but longer codes.

---

## Scaling Write Path

If URL creation becomes a bottleneck:
- Horizontal scale the URL Service — stateless, easy to scale
- Use a distributed ID generator (Twitter Snowflake) instead of DB auto-increment
- Shard PostgreSQL by user_id if needed

---

## Full Architecture Summary

```
Client
 ↓
CDN (cached redirects for popular URLs)
 ↓
API Gateway (rate limiting, auth)
 ↓
URL Service (stateless, horizontally scaled)
 ↓               ↓
Redis Cache    PostgreSQL Primary
(short→long)        ↓
                Read Replica
                (redirect reads)
```

---

## Topics Covered in This Design

| Topic | Concept |
|---|---|
| Hashing | MD5 vs Base62 encoding |
| Unique ID generation | Auto-increment, distributed counter, Snowflake |
| Database design | Schema, indexing on short_code |
| Read scaling | Redis cache, Read Replica, CDN |
| Caching | Cache aside, TTL, 99% hit rate target |
| Rate limiting | Redis counter, 429 response |
| Redirect types | 301 permanent vs 302 temporary |
| CDN | Edge caching for popular redirects |

---

## Key Interview Phrases

- "The read to write ratio is 100:1 so the design is optimised for reads — Redis cache and CDN handle 99% of redirects without hitting the database"
- "We use Base62 encoding on a unique auto-increment ID — 6 characters gives us 56 billion unique codes with zero collision risk"
- "301 redirect is cached by the browser so it never hits our server again — great for performance but kills click analytics. 302 ensures every click is tracked."
- "CDN caches the most popular redirects at edge nodes — sub 5ms response without any server involvement"
- "Rate limiting at API Gateway prevents abuse — Redis counter with 1 hour TTL, return 429 after threshold"
