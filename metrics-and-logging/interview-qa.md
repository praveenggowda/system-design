# Metrics and Logging — Mock Interview Q&A

## Clarification Phase

**Interviewer:** Design a system that collects metrics and logs from hundreds of microservices and allows engineers to query them in real time. Go.

**Praveen:** Before I start — a few clarifying questions.
- The clients here are microservices sending metrics and logs — is that correct?
- How many events are we expecting per second?
- Is the data structured or unstructured?
- What is the query latency requirement?
- Does the system need to acknowledge receipt back to the microservice?
- What kind of metrics and logs are we storing — errors, latency, success rates?
- How long do we retain data?
- Who queries the data and how?

**Interviewer:** 500 microservices each sending 1,000 events/sec — 500,000 events/sec at peak. Mix of structured metrics and semi-structured logs. Logs queryable within 10 seconds, metrics within 5 seconds. Fire and forget — no acknowledgement. Errors, latency, CPU/memory, HTTP status codes. Logs retained 30 days, metrics 1 year. Engineers query via dashboard (Grafana) and ad-hoc search.

---

## Back of Envelope

- Assumption: average traffic is 10% of peak — microservices have spiky patterns
- Average: 500,000 * 10% = 50,000 events/sec
- Daily volume: 50,000 * 86,400 = **4.32B events/day**
- 1 event = ~1KB
- Daily storage: 4.32B * 1KB = **4.32TB/day**
- Annual storage: 4.32TB * 365 = **~1.58PB/year**
- Peak throughput: 500,000 * 1KB = **500MB/sec**

**Design implication:** Data volume is enormous. Cannot use a single relational database. Need specialised storage: time-series DB for metrics, search engine for logs. Need a queue to absorb peak bursts before writing to storage.

---

## Non-Functional Requirements

- High write throughput — 500,000 events/sec at peak
- Low query latency — logs within 10 seconds, metrics within 5 seconds
- AP system — availability over consistency. A missing metric for a few seconds is acceptable. Dropped logs are not ideal but preferable to blocking services.
- Retention: logs 30 days, metrics 1 year
- Cost-efficient at petabyte scale

---

## High Level Design

**Praveen:** At 500,000 events/sec, HTTP calls from each microservice per event would overwhelm the API Gateway. Instead, each microservice runs a **log agent** (Fluent Bit) as a sidecar that batches events locally every few seconds and sends them in bulk.

The Ingestion Service reads the `type` field from each event and routes to one of two Kafka topics:
- `type: "metric"` → **metrics-topic**
- `type: "log"` → **logs-topic**

Two separate consumers read from their respective topics:
- **Metrics Consumer** → writes to **InfluxDB** (time-series database)
- **Logs Consumer** → writes to **Elasticsearch** (full-text search)

Engineers query via:
- **Grafana** → queries InfluxDB for dashboards, time-series graphs, and alerting
- **Kibana** → queries Elasticsearch for log search and error drilling

**Interviewer:** Why Kafka instead of SQS?

**Praveen:** At 500,000 events/sec, Kafka is purpose-built for this scale. It also allows multiple consumers to read the same stream independently — metrics consumer and logs consumer both read from Kafka without competing. SQS is point-to-point; once a message is consumed it is gone.

**Interviewer:** The event has a type field — who sets it?

**Praveen:** The emitting microservice sets it. The Ingestion Service reads the field and routes — no complex logic needed. The responsibility for correct tagging sits with the emitting service.

---

## Cost and Storage Tiering

**Interviewer:** At 4.32TB/day for 30 days, that is 130TB of log storage. Elasticsearch is expensive at that scale. How do you manage cost?

**Praveen:** Tiered storage:

```
Day 0-7   → Elasticsearch (hot — fast, queryable, expensive)
Day 7-30  → S3 (cold — compressed, cheap)
Day 30+   → deleted automatically
```

After 7 days, logs are archived to S3 in compressed format. If engineers need older logs they can query S3 with Amazon Athena. Auto-delete from S3 after 30 days.

For metrics — InfluxDB has built-in downsampling. After 7 days, raw per-second metrics are aggregated into hourly averages. This reduces storage by orders of magnitude while still allowing trend analysis over a year.

---

## Backpressure

**Interviewer:** You cannot use rate limiting here — you need all the events. How do you handle the write load?

**Praveen:** Kafka absorbs the burst. At peak, Elasticsearch may only handle 50,000 writes/sec but we receive 500,000/sec. Kafka buffers the difference and the consumer catches up when load drops. The storage systems never get overwhelmed because they receive events at their own sustainable rate from the consumer — not directly from the microservices.

---

## Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Ingestion | Fluent Bit agent + batching | HTTP per-event at 500k/sec overwhelms API Gateway |
| Queue | Kafka | High throughput, multiple independent consumers, message retention |
| Metrics storage | InfluxDB | Time-series optimised, built-in downsampling, fast range queries |
| Log storage | Elasticsearch | Full-text search, structured JSON queries, Kibana integration |
| Routing | type field on event | Simple, microservice owns the classification |
| Cost management | Tiered storage to S3 after 7 days | Elasticsearch expensive at petabyte scale |
| Metrics retention | Downsampling after 7 days | Raw per-second data aggregated to hourly averages |
| Alerting | Grafana alert rules | Polls InfluxDB, fires to PagerDuty/Slack on threshold breach |
