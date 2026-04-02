# 32. Metrics & Monitoring — Prometheus, Grafana, Datadog

## What are Metrics?

Metrics are **numeric measurements collected over time** — the pulse of your system. Unlike logs (discrete events) or traces (per-request journeys), metrics are aggregated and efficient to store, making them ideal for dashboards, alerts, and capacity planning.

**Examples:**
- `http_requests_total{service="payment", status="500"}` = 42
- `db_query_duration_seconds{quantile="0.99"}` = 0.8
- `cache_hit_ratio` = 0.94

---

## Q1: What are the four metric types?

### 1. Counter
Monotonically increasing value. Only goes up (or resets to 0 on restart).

```
http_requests_total = 1,042,891
errors_total = 234
```

Use for: request counts, error counts, bytes sent. Query with `rate()` to get per-second rate.

### 2. Gauge
A value that can go up or down. Snapshot of current state.

```
active_connections = 142
memory_used_bytes = 2,147,483,648
queue_depth = 33
```

Use for: current resource usage, queue depth, temperature.

### 3. Histogram
Samples observations into configurable **buckets**. Lets you compute percentiles.

```
http_request_duration_seconds_bucket{le="0.1"} = 8412   (requests < 100ms)
http_request_duration_seconds_bucket{le="0.5"} = 9891   (requests < 500ms)
http_request_duration_seconds_bucket{le="1.0"} = 9994   (requests < 1s)
http_request_duration_seconds_bucket{le="+Inf"} = 10000 (all requests)
http_request_duration_seconds_sum = 1234.5
http_request_duration_seconds_count = 10000
```

Use for: latency distributions, request size distributions. Needed to compute P50/P95/P99.

### 4. Summary
Pre-computes percentiles client-side. Less flexible than histograms (can't aggregate across instances).

**In practice:** Prefer histograms over summaries — histograms can be aggregated across multiple service instances in Prometheus.

---

## Q2: What are the RED and USE methods?

These are frameworks for deciding *what to measure*.

### RED Method (for services/APIs)
Coined by Tom Wilkie. For every service, track:

| Metric | Description | Example |
|--------|-------------|---------|
| **R**ate | Requests per second | `rate(http_requests_total[5m])` |
| **E**rrors | Error rate (4xx/5xx) | `rate(http_errors_total[5m])` |
| **D**uration | Latency distribution | P50, P95, P99 of request duration |

RED is user-facing — it tells you if your service is serving users well.

### USE Method (for resources/infrastructure)
Coined by Brendan Gregg. For every resource (CPU, memory, disk, network):

| Metric | Description | Example |
|--------|-------------|---------|
| **U**tilization | % of time resource is busy | CPU usage: 73% |
| **S**aturation | Queue/backlog building up | Run queue length > 1 = saturated |
| **E**rrors | Error events | Disk I/O errors, network packet drops |

USE tells you if your infrastructure is the bottleneck.

**Together:** RED tells you *users are experiencing slowness*. USE tells you *why* (CPU saturated, disk I/O errors, etc.).

---

## Q3: How does Prometheus work?

**Prometheus** is the de facto open-source metrics system. It uses a **pull model** — Prometheus scrapes metrics from services on a schedule.

### Architecture

```
Services expose → /metrics endpoint (text format)
                         ↑
              Prometheus scrapes every 15s
                         │
              Stores in TSDB (Time Series DB)
                         │
              PromQL queries → Grafana / Alertmanager
```

### Prometheus Metric Format (text exposition)

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="500"} 5
```

### PromQL (Prometheus Query Language)

```promql
# Request rate per second (last 5 min)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# P99 latency from histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU usage per pod
100 - (avg by (pod) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Labels

Labels are key-value pairs that add dimensions to metrics. They're powerful but dangerous:
- `{service="payment", env="prod", method="POST"}` — good
- `{user_id="12345"}` — **bad** — high cardinality (millions of users = millions of time series → OOM)

**Rule:** Never use unbounded values as labels (user_id, request_id, URL path with IDs).

### Pushgateway
For short-lived jobs (cron, batch) that finish before Prometheus can scrape them, they push metrics to the Pushgateway, which Prometheus then scrapes.

---

## Q4: What is Grafana?

**Grafana** is the visualization layer — it queries Prometheus (and other data sources) and renders dashboards.

### Key Features
- **Dashboards:** Panels with graphs, gauges, tables, heatmaps
- **Multiple data sources:** Prometheus, Loki (logs), Jaeger (traces), MySQL, Elasticsearch — all in one UI
- **Alerting:** Alert rules on any metric query
- **Variables:** Dashboard templates (`$service`, `$env`) — one dashboard for all services
- **Annotations:** Mark deployments on graphs to correlate with latency changes

### Golden Signals Dashboard (SRE standard)

Four panels every service should have:
1. Request rate (RPS)
2. Error rate (%)
3. Latency (P50/P95/P99)
4. Saturation (CPU/memory/queue depth)

---

## Q5: What is Datadog and how does it differ from Prometheus + Grafana?

**Datadog** is a commercial, fully-managed observability platform.

| | Prometheus + Grafana | Datadog |
|---|---|---|
| Hosting | Self-managed | Managed SaaS |
| Cost | Infrastructure only | Per-host + per-custom-metric |
| Metrics | Yes (Prometheus) | Yes (Agent) |
| Logs | Separate (Loki/ELK) | Native |
| Traces (APM) | Separate (Jaeger) | Native |
| Correlation (metrics↔logs↔traces) | Manual setup | Native, one click |
| Alerting | Alertmanager | Built-in |
| ML anomaly detection | No | Yes |
| Setup time | Days-weeks | Hours |
| Scale ceiling | High (Thanos/Cortex) | Very high (managed) |

**When to use Datadog:** Teams that want a single pane of glass, don't want to manage infrastructure, willing to pay for convenience.

**When to use Prometheus + Grafana:** Cost-sensitive, want full control, open-source preference, large scale where Datadog costs are prohibitive.

---

## Q6: How do you scale Prometheus?

Prometheus stores data locally — a single instance handles ~10M active time series. Beyond that:

**Thanos:**
- Adds global query view across multiple Prometheus instances
- Long-term storage in S3/GCS
- Deduplication of HA Prometheus pairs

**Cortex / Mimir (Grafana):**
- Horizontally scalable, multi-tenant Prometheus backend
- Remote write from Prometheus → Cortex cluster
- Stores in object storage

```
Prometheus A → [remote_write] → Cortex/Mimir (distributed TSDB)
Prometheus B → [remote_write] →              ↓
                                    Grafana (global queries)
```

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Prometheus scrape interval | 15s (default) |
| Prometheus TSDB retention | 15 days (default) |
| Single Prometheus max time series | ~10M |
| Datadog agent CPU overhead | ~1-3% |
| Metric cardinality limit (Prometheus) | ~1M unique label combos per metric |
| Grafana dashboard refresh rate | 10s–5min (typical) |
| PromQL range for rate() | Minimum 2× scrape interval (e.g., [30s]) |

---

## Interview Q&A

**Q: Your P99 latency is 2 seconds but P50 is 50ms. What does this tell you and how do you investigate?**
A: A huge gap between P50 and P99 means most requests are fast but a small fraction (1%) are very slow. Common causes: database slow queries, garbage collection pauses, a downstream service timing out for some requests, hot cache keys causing lock contention. I'd look at histogram buckets to understand the distribution, check if the slow P99 correlates with specific endpoints/users/shards, and use distributed tracing to find which span accounts for the extra ~1950ms in those slow requests.

**Q: Why is high-cardinality labeling dangerous in Prometheus?**
A: Each unique label combination creates a separate time series. `user_id` with 10M users × 5 metrics = 50M time series. Prometheus loads all active time series into memory — this causes OOM crashes or extreme slowness. Also, cardinality explosion makes queries slow. Solution: never use user_id, request_id, or URL parameters as labels. Aggregate in the application before emitting metrics (e.g., track `payment_failed_total` by `reason`, not by `user_id`).

**Q: What's the difference between monitoring and observability?**
A: Monitoring is about tracking known failure modes — dashboards and alerts for things you anticipate could go wrong. Observability is about being able to ask arbitrary questions about system behavior, including failures you didn't predict. A system is observable if you can understand its internal state from its external outputs (metrics, logs, traces) without deploying new code. Observability requires structured telemetry; monitoring requires thresholds. Both are necessary — monitoring for fast alerting on known issues, observability for diagnosing novel problems.