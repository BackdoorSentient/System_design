# 31. Logging at Scale — Structured Logging & Log Aggregation

## What is Logging at Scale?

A single service writing logs to a file is simple. At scale — hundreds of services, thousands of instances, millions of requests per second — you need logs to be:
- **Structured** (machine-readable, not just free text)
- **Centralized** (aggregated from all instances into one queryable store)
- **Searchable** (find the one log line for user X's failed request in seconds)
- **Retained and archived** (compliance, debugging old issues)

---

## Q1: What is structured logging and why does it matter?

**Unstructured log (bad for scale):**
```
2024-01-15 10:23:45 ERROR Payment failed for user 12345 amount 99.99 reason card_declined
```

You'd have to use `grep` with regex to parse this. Impossible to query efficiently at scale, impossible to aggregate metrics from.

**Structured log (JSON):**
```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "event": "payment_failed",
  "user_id": 12345,
  "amount": 99.99,
  "reason": "card_declined",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "request_id": "req-abc-123",
  "host": "payment-pod-7d9f4",
  "duration_ms": 145
}
```

Now you can:
- Query: `event=payment_failed AND reason=card_declined` (last hour)
- Aggregate: Count payment failures by reason per 5 minutes → alert
- Correlate: Jump from this log to the Jaeger trace via `trace_id`

**Structured logging libraries:**

| Language | Library |
|----------|---------|
| Python | `structlog`, `python-json-logger` |
| Go | `zerolog`, `zap` |
| Java | Logback + `logstash-logback-encoder` |
| Node.js | `pino`, `winston` |

---

## Q2: What standard fields should every log line include?

| Field | Why |
|-------|-----|
| `timestamp` (ISO 8601, UTC) | Time-based queries, correlation |
| `level` (DEBUG/INFO/WARN/ERROR) | Filter noise |
| `service` | Which microservice |
| `trace_id` | Correlate with distributed trace |
| `request_id` / `correlation_id` | Trace one user request |
| `user_id` / `tenant_id` | Debug user-specific issues |
| `host` / `pod_name` | Which instance had the problem |
| `event` | What happened (machine-readable name) |
| `duration_ms` | Performance data inline |
| `error` (with stack trace) | On ERROR level |

---

## Q3: What is the ELK Stack and how does it work?

**ELK = Elasticsearch + Logstash + Kibana**

```
Services → [Log Shipper] → [Logstash/Kafka] → [Elasticsearch] → [Kibana]
```

### Elasticsearch
- Distributed search and analytics engine
- Stores logs as JSON documents in **indices** (typically one per day: `logs-2024-01-15`)
- Full-text search + structured queries
- **Inverted index** — fast field-level search even on billions of documents

### Logstash
- Ingestion pipeline: receives logs, parses/transforms, forwards to Elasticsearch
- Supports grok patterns (parse unstructured text into structured fields)
- Can enrich logs (add geo-IP, lookup metadata)
- **Bottleneck risk:** single-threaded pipeline. Often replaced by Kafka as buffer.

### Kibana
- Web UI for querying, visualizing, and dashboarding Elasticsearch data
- **KQL (Kibana Query Language):** `service: "payment" and level: "ERROR"`
- Dashboards: log volume, error rates, top errors
- Discover tab: real-time log tail

### Beats (lightweight shippers)
- **Filebeat:** Tails log files, ships to Logstash or Elasticsearch directly
- **Metricbeat:** System metrics
- Runs as a sidecar or DaemonSet on every node

---

## Q4: What is the Elastic vs Loki trade-off?

**Grafana Loki** is a newer, cost-efficient alternative to Elasticsearch for logs.

| | Elasticsearch | Loki |
|---|---|---|
| Indexing | Full index on all fields | Index only labels (metadata) |
| Storage cost | High (index = 10–30% of raw log size) | Low (compressed raw logs) |
| Query speed | Very fast (pre-indexed) | Slower (full scan with label filter) |
| Query language | KQL / Lucene | LogQL (similar to PromQL) |
| Best for | Complex search, high cardinality | Label-based filtering, cost-sensitive |
| Integrates with | Kibana | Grafana |

**Loki's approach:** Only index a few high-cardinality-safe labels (service, environment, pod). The log body is stored compressed and scanned at query time. For most operational queries ("show me ERROR logs from payment-service in the last hour"), label filtering gets you to a small enough set that full-scan is fast.

**When to use Elasticsearch:** Complex search (full-text, fuzzy), many different query patterns, compliance use cases with rich querying needs.

**When to use Loki:** Cost-sensitive, Kubernetes-native, Grafana is already your metrics dashboard, you query by service/pod/namespace.

---

## Q5: What is a log aggregation pipeline at scale?

At high volume (>1M logs/sec), a naive "ship directly to Elasticsearch" approach breaks. You need a buffer.

```
Services
   │
   ▼ (Filebeat / Fluentd sidecars)
[Kafka Topic: logs]          ← durable buffer, survives Elasticsearch downtime
   │
   ▼ (Logstash / custom consumer)
[Elasticsearch cluster]
   │
   ▼
[Kibana]          [S3 archive]  ← cold storage for old logs
```

**Why Kafka as a buffer?**
- Elasticsearch goes down for maintenance → logs don't drop, Kafka retains them
- Traffic spike → Kafka absorbs burst, consumers drain at their own pace
- Multiple consumers: one writes to Elasticsearch, one to S3 archive, one triggers alerts on ERROR patterns

---

## Q6: Log levels — what goes where?

| Level | Use | Volume |
|-------|-----|--------|
| `DEBUG` | Detailed internal state, variable values | Very high — only in dev/staging |
| `INFO` | Normal business events ("Order placed", "User logged in") | Medium |
| `WARN` | Unexpected but recoverable ("Retry #2", "Cache miss rate high") | Low |
| `ERROR` | Failed operation that affects functionality | Very low |
| `FATAL` / `CRITICAL` | Process is about to crash | Extremely rare |

**Production rule:** INFO and above. DEBUG only with dynamic log-level changing (change level without restart) for targeted debugging sessions.

---

## Q7: Log retention and cost management

| Age | Tier | Storage |
|-----|------|---------|
| 0–7 days | Hot | Elasticsearch SSD (fast search) |
| 7–30 days | Warm | Elasticsearch HDD (slower search) |
| 30–90 days | Cold | S3 / GCS (download to query) |
| 90+ days | Archive | S3 Glacier / Coldline (compliance only) |

**ILM (Index Lifecycle Management) in Elasticsearch** automates this tiering.

**Cost saving tips:**
- Drop DEBUG logs in production (filter in Logstash before writing to ES)
- Sample INFO logs (keep 10%, keep 100% of WARN+)
- Compress before archiving (logs compress 10–20:1 with gzip)

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Average log line size (structured JSON) | 500 bytes – 2 KB |
| Elasticsearch index overhead | ~10–30% of raw log size |
| Loki compression ratio | 10:1 to 20:1 |
| Typical log volume (busy microservice) | 1,000–10,000 lines/sec |
| Kafka retention for log buffer | 24–48 hours |
| Elasticsearch hot tier query latency | <100ms for indexed fields |

---

## Interview Q&A

**Q: How do you find the logs for one specific user's failed request across 10 microservices?**
A: Every log line includes a `request_id` (or `trace_id`) generated at the API gateway and propagated via headers through all downstream calls. Search Kibana/Loki for `request_id: "req-abc-123"` — you get all logs from all services for that one request, ordered by timestamp. This is why structured logging with a correlation ID is non-negotiable at scale.

**Q: Why not just keep all logs in Elasticsearch forever?**
A: Storage cost. A busy system can generate terabytes of logs per day. Elasticsearch on SSD is expensive — you'd pay 10–100× more than cold storage for logs nobody reads after 7 days. ILM tiering (hot → warm → cold → archive) gives you fast queries on recent logs while keeping old logs cheap. Regulatory requirements (GDPR, SOC2) also mandate specific retention windows, not infinite.

**Q: A developer left a `logger.debug()` call that logs the full request body including passwords. How do you handle this in a logging pipeline?**
A: Defense in layers. In the application: never log sensitive fields — use an allowlist of loggable fields. In the pipeline (Logstash/OTel collector): add a filter to redact known sensitive field names (`password`, `token`, `credit_card`) before writing to storage. In Elasticsearch: field-level security to restrict who can query sensitive fields. Audit log access. This is a people+process problem as much as a technical one — add sensitive data scanning to code review.