# Search System — Interview Q&A

---

## Clarifications

- Full-text search across item names, column values, comments, and updates
- Generic search — not column-specific, no restriction on search string
- Input validation: sanitise input, strip HTML tags, cap at 200 characters
- API: `GET /search?q=searchstring&page=1&limit=20` — query params, not path segments
- Pagination: results returned per page, load next page on demand
- Permissions: every result must be filtered to boards the requesting user has access to — non-negotiable
- Newly created or updated items must appear in search within 2-3 seconds
- AP system: eventual consistency on data is acceptable (new items may take seconds to appear), strict consistency on permissions
- Scale: 5000 RPS peak, 50 million items across all boards
- Caching: search results can be cached but permission filtering must always be applied at serve time
- Write storage is out of scope — this system only covers the search read path and indexing pipeline

---

## Back of Envelope

```
Peak:    5000 RPS × 100K seconds = 500M requests/day
Average: 500 RPS × 100K seconds  = 50M requests/day
  (search is bursty — peaks during business hours)

Response size: ~50KB per page of 20 results
Throughput:    5000 × 50KB = 250 MB/s outbound at peak

Items: 50M total
Index size: 50M × ~2KB per document = ~100GB in Elasticsearch

Sanity check: 500K boards × 10 active users = 5000 concurrent requests ✓
```

---

## High-Level Architecture

```
User types search string
  ↓
GET /search?q=payment&page=1&limit=20
  ↓
API Gateway (auth, rate limiting)
  ↓
Search Service
  ├── Redis Cache (query-level cache, raw results — not per-user)
  │   HIT  → apply permission filter for requesting user → return
  │   MISS → query Elasticsearch
  ↓
Elasticsearch
  ├── Full-text match on content fields
  └── Permission filter (allowedUsers, allowedTeams in document)
  ↓
Search Service applies final permission check + pagination
  ↓
Return results to user

--- Indexing Pipeline ---

Board Item Created/Updated (POST or PATCH)
  ↓
Board Service
  ├── PostgreSQL Primary
  │     BEGIN TRANSACTION
  │       INSERT/UPDATE items
  │       INSERT INTO outbox (event_type, payload)
  │     COMMIT
  ↓
Outbox Worker → Kafka (partitioned by boardId + itemId)
  ↓
Search Indexer (Kafka consumer)
  ↓
Elasticsearch (upsert by documentId = itemId)
```

---

## Elasticsearch Document Model

```json
{
  "_id": "item-123",
  "documentId": "item-123",
  "boardId": "board-456",
  "title": "Fix payment bug",
  "content": "Fix payment bug description and all column values concatenated",
  "updatedAt": "2026-07-20T10:00:00Z",
  "allowedUsers": ["user-a", "user-b"],
  "allowedTeams": ["team-30", "team-45"]
}
```

Permissions are embedded in the document. Permission filtering happens at query time inside Elasticsearch — not in the application layer after fetching.

---

## Elasticsearch Query

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "content": "payment" } }
      ],
      "filter": [
        {
          "bool": {
            "should": [
              { "term": { "allowedUsers": "user-a" } },
              { "terms": { "allowedTeams": ["team-30", "team-45"] } }
            ]
          }
        }
      ]
    }
  },
  "from": 0,
  "size": 20
}
```

`must` — full-text match, contributes to relevance score.
`filter` — permission check, no scoring, cached by Elasticsearch, fast.

---

## Initial Backfill

When Elasticsearch is first introduced, existing data must be indexed from scratch. Do not run the bulk indexer against the PostgreSQL Primary — that adds heavy read load to the operational database.

```
PostgreSQL Read Replica
  ↓
Bulk Indexer (reads in batches of 10,000 rows)
  ↓
Elasticsearch Bulk API
```

After backfill completes, ongoing updates flow through the normal Outbox → Kafka → Search Indexer path.

---

## Idempotent Indexing

The Search Indexer uses the item's ID as the Elasticsearch document `_id`. Every write is an upsert:

```
PUT /items/_doc/item-123
{ ... document ... }
```

If the Kafka consumer fails to commit its offset after writing to Elasticsearch and reprocesses the same message, Elasticsearch overwrites the existing document — no duplicate.

Ordering is guaranteed because Kafka is partitioned by `boardId + itemId`. Same item's events always land in the same partition, consumed in order.

---

## Permission Filtering — Two Approaches

**Approach 1: Scope-based (preferred at scale)**

Resolve the user's allowed board/workspace IDs from an authorization service, then pass them as a filter in the Elasticsearch query:

```json
"filter": [
  { "terms": { "boardId": ["board-100", "board-200", "board-300"] } }
]
```

Simpler, cheaper, no per-document ACL maintenance. Works when access is controlled at board or workspace level.

**Approach 2: Per-record ACL (for fine-grained access)**

Embed `allowedUsers` and `allowedTeams` in each document. Handles item-level permissions but requires updating every document when permissions change. Use `update_by_query` for bulk updates (see below).

For monday.com: start with scope-based. Fall back to per-record ACL only if item-level permissions are required.

**Never filter in the application layer after fetching.** Fetching results then discarding unauthorised ones causes incorrect pagination counts, wastes Elasticsearch throughput, and risks leaking document metadata.

---

## Permission Updates — Bulk Update Problem

When a user is removed from a board with 10,000 items:

```
Permission change event → Kafka
  ↓
Search Indexer consumes it
  ↓
Elasticsearch update_by_query:
  Update all documents WHERE boardId = X
    → remove "user-a" from allowedUsers
```

`update_by_query` runs as a throttled background operation in Elasticsearch. Live search traffic is not blocked. The window where stale permissions exist is seconds to minutes — acceptable for a collaboration tool.

Alert if `update_by_query` takes longer than expected — a 10-minute window to remove access is a security concern.

---

## Pagination

For small result sets use `from + size`:
```json
{ "from": 0, "size": 20 }
```

For deep pagination (page 500+), `from + size` forces Elasticsearch to score and skip millions of records — very expensive.

Use `search_after` with a stable sort key instead:
```json
{
  "sort": [{ "createdAt": "asc" }, { "documentId": "asc" }],
  "search_after": ["2026-01-01T00:00:00Z", "item-4999"]
}
```

Each page returns a cursor (the last sort values). The next page uses that cursor. No offset scanning.

---

## Caching Strategy

Cache at the query level, not the user level.

```
Cache key: hash(q, page, limit)
Value:     raw Elasticsearch result set (all matching documents including allowedUsers/allowedTeams)
TTL:       30 seconds
```

When serving from cache: apply permission filter for the requesting user from the cached result set. User A and User B searching the same query hit the same cache entry but see different results based on their permissions.

Never cache per-user results — that would require a separate cache entry per user per query, making caching largely ineffective.

---

## Failure Scenarios

| Failure | Behaviour |
|---|---|
| Search Indexer fails to commit Kafka offset | Reprocesses same message — Elasticsearch upsert is idempotent, no duplicate |
| Elasticsearch node down | Elasticsearch cluster rebalances across remaining nodes. Search degrades in latency but remains available. |
| Kafka consumer lag grows | Indexing falls behind — new/updated items take longer to appear in search. Alert on consumer lag. |
| Redis cache unavailable | Fall through to Elasticsearch directly. Higher latency, Elasticsearch absorbs full load. |
| Outbox Worker down | Items created during downtime queue up in Outbox table. Worker picks them up on recovery. No data lost. |
| Permission update `update_by_query` slow | Alert if taking longer than threshold. Temporarily a removed user may still see results — acceptable window, not a data breach. |

---

## Observability

**Logs**
Structured logs: board_id, item_id, user_id (name only, no email), event_type, timestamp, trace_id. Flows from API Gateway through Search Service → Elasticsearch → Search Indexer.

**Metrics**

| Metric | Why |
|---|---|
| Search latency p99 | Target under 200ms — drop means Elasticsearch degraded |
| Zero-result rate | Spike = index broken or items not indexed yet |
| Elasticsearch indexing lag | Time from item write to appearing in search — target under 3 seconds |
| Cache hit rate | Low rate = cache not warming or TTL too short |
| Kafka consumer lag | Growing = Search Indexer falling behind |
| Outbox depth | Growing = Outbox Worker down |
| `update_by_query` duration | Long duration = permission staleness risk |
| RPS at Search Service | Spike = load event or abuse |

**Alerts**
- Indexing lag > 10 seconds
- Zero-result rate spike
- Kafka consumer lag > 10K messages
- `update_by_query` duration > 5 minutes
- Cache hit rate < 30%

Route alerts to Grafana / Datadog / Slack.

---

## Trade-offs

| Decision | Why | Alternative | Cost |
|---|---|---|---|
| Embed permissions in Elasticsearch document | Single query handles search + permission filtering. Fast, no extra round trips. | Filter in application layer after fetching | Fetch more documents than needed, discard unauthorised ones. Wastes Elasticsearch throughput. Risks leaking metadata. |
| CDC via Outbox → Kafka → Search Indexer | Decoupled, reliable, Outbox guarantees no event lost. Elasticsearch updated within seconds. | Query PostgreSQL directly on every search | Full-text search on PostgreSQL degrades under 5000 RPS on 50M items. No relevance scoring. Index on all columns is impractical. |
| Query-level cache, permission filter at serve time | Cache is shared across users — high hit rate. Permission filtering is cheap from cached result set. | Per-user cache | One cache entry per user per query — effectively no reuse, defeats the purpose of caching |
| Elasticsearch over PostgreSQL full-text | Inverted index, relevance scoring, fuzzy matching, typo tolerance, horizontal scaling. Purpose-built for search. | PostgreSQL tsvector + GIN index | Works at small scale, degrades under high concurrent read load and complex multi-field queries |
| Partition Kafka by boardId + itemId | Guarantees ordered processing per item — no stale overwrites | Partition by boardId only | Multiple items on same board still processed in order but higher partition contention |

---

## Diagram

See `search-system.excalidraw`.
