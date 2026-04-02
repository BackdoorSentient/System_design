# 30. Distributed Tracing — OpenTelemetry & Jaeger

## What is Distributed Tracing?

In a microservices system, a single user request may pass through 10–20 services. When something is slow or broken, logs from individual services don't tell you the full story — you need to see the **end-to-end journey** of that request.

Distributed tracing reconstructs the complete path of a request across all services, showing:
- Which services were called, in what order
- How long each hop took
- Where errors or latency spikes occurred

---

## Q1: What are Traces, Spans, and Trace Context?

### Trace
A **trace** represents the full lifecycle of one request across all services. It has a globally unique **Trace ID**.

### Span
A **span** is a single unit of work within a trace — one service call, one DB query, one external API call. It has:
- **Span ID** — unique within the trace
- **Parent Span ID** — links to the calling span
- **Start time + duration**
- **Tags/attributes** — key-value metadata (HTTP method, status code, DB query)
- **Events/logs** — timestamped annotations within the span

```
Trace ID: abc-123
│
├── Span: API Gateway (0ms → 250ms)
│     ├── Span: Auth Service (5ms → 20ms)
│     ├── Span: Order Service (25ms → 200ms)
│     │     ├── Span: DB Query - orders (30ms → 80ms)
│     │     ├── Span: Inventory Service (85ms → 150ms)
│     │     └── Span: Payment Service (155ms → 195ms)
│     └── Span: Notification Service (205ms → 245ms)
```

This visual is called a **waterfall diagram** — the core output of a tracing tool like Jaeger.

### Trace Context Propagation
For a trace to work across services, the Trace ID and Span ID must be **passed in every request header**.

**W3C TraceContext standard (recommended):**
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             version  traceId                          spanId           flags
```

Each service reads this header, creates a child span with its own Span ID, and passes the updated header downstream.

---

## Q2: What is OpenTelemetry?

**OpenTelemetry (OTel)** is the industry-standard open-source framework for generating, collecting, and exporting telemetry data (traces, metrics, logs). It's vendor-neutral — you instrument once and export to Jaeger, Datadog, Honeycomb, Zipkin, or any backend.

### Components

**SDK (in your application):**
- Libraries for Python, Go, Java, Node.js, etc.
- Auto-instrumentation: patches popular frameworks (Flask, FastAPI, Django, Express, gRPC) automatically — zero code changes
- Manual instrumentation: create custom spans for business logic

**Collector (standalone agent/sidecar):**
- Receives telemetry from applications
- Processes (batch, filter, sample)
- Exports to one or more backends

```
App (OTel SDK)
    │
    │ OTLP (gRPC or HTTP)
    ▼
OTel Collector ──► Jaeger (traces)
               ──► Prometheus (metrics)
               ──► Loki (logs)
```

**OTLP** (OpenTelemetry Protocol) is the standard wire protocol for sending telemetry data.

### Auto-instrumentation Example (Python)

```python
# Zero-code instrumentation — just configure at startup
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument()
# Every HTTP request and DB query now emits spans automatically
```

### Manual Instrumentation Example

```python
from opentelemetry import trace

tracer = trace.get_tracer("order-service")

def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.value", get_order_value(order_id))
        
        result = charge_payment(order_id)
        if result.failed:
            span.set_status(StatusCode.ERROR, "Payment failed")
            span.record_exception(result.exception)
        return result
```

---

## Q3: What is Jaeger?

**Jaeger** is an open-source distributed tracing backend, originally built by Uber. It stores and visualizes traces.

### Architecture

```
Application → [Jaeger Agent] → [Jaeger Collector] → [Storage: Cassandra/Elasticsearch]
                                                              │
                                                    [Jaeger Query] → [Jaeger UI]
```

- **Jaeger Agent:** Sidecar that receives spans from the application (UDP, low overhead) and forwards to collector
- **Jaeger Collector:** Validates, indexes, and stores spans
- **Storage:** Cassandra (for high write throughput) or Elasticsearch (for powerful querying)
- **Jaeger UI:** Web interface — trace search, waterfall diagrams, service dependency graphs

### Key Features
- Trace search by service, operation, tags, duration, error status
- Dependency graph — shows which services call which (auto-generated from trace data)
- Comparison — compare two traces side by side
- Service performance monitoring (P50/P95/P99 per operation)

---

## Q4: What is sampling and why is it necessary?

**The problem:** A high-traffic service handling 100,000 req/sec would generate millions of spans per second. Storing and processing all of them is prohibitively expensive.

**Sampling** = only record a subset of traces.

### Head-based sampling (most common)
Decision made at the start of the request (at the first service).

- **Probabilistic:** Record X% of all requests (e.g., 1%)
- **Rate limiting:** Record N traces per second regardless of load
- **Always-on for errors:** Always record traces that contain errors

**Problem:** You might sample out the one slow trace you needed to debug.

### Tail-based sampling (smarter, harder)
Decision made **after** the request completes — collect all spans, then decide which to keep.

- Keep all traces with errors
- Keep all traces above P99 latency threshold
- Sample the rest at 1%

**Problem:** Requires buffering all spans before making the decision — more infrastructure.

**Used by:** Honeycomb, newer OTel collector processors

### Adaptive sampling
Automatically adjusts sample rate per service/operation to maintain a target rate. High-traffic endpoints sampled less; rare endpoints sampled more.

---

## Q5: Jaeger vs Zipkin vs Datadog APM

| | Jaeger | Zipkin | Datadog APM |
|---|---|---|---|
| Open source | Yes | Yes | No (commercial) |
| Origin | Uber | Twitter | Datadog |
| Storage | Cassandra, Elasticsearch | MySQL, Cassandra, Elasticsearch | Managed |
| OTel support | Yes | Yes | Yes |
| UI quality | Good | Basic | Excellent |
| Alerting on traces | No (need separate tool) | No | Yes |
| Correlation with metrics/logs | Manual | Manual | Native |
| Cost | Infrastructure only | Infrastructure only | Per-host/per-span pricing |

---

## Q6: How do you correlate traces with logs and metrics?

The **three pillars of observability** — traces, metrics, logs — are most powerful when linked.

**Trace ID in logs:**
```python
import logging
from opentelemetry import trace

def process_order(order_id):
    current_span = trace.get_current_span()
    trace_id = format(current_span.get_span_context().trace_id, '032x')
    
    logger.info("Processing order", extra={
        "trace_id": trace_id,
        "order_id": order_id
    })
```

Now you can jump from a slow Jaeger trace directly to the logs for that exact request.

**Exemplars (metrics → traces):**
Prometheus supports exemplars — attaching a Trace ID to a specific metric data point. When you see a latency spike on a Grafana chart, click the spike to jump to the Jaeger trace that caused it.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Typical span size | 1–5 KB |
| Jaeger storage (1% sampling, 10k RPS) | ~50–200 GB/day |
| OTel auto-instrumentation overhead | <1% CPU, <5ms latency added |
| Trace context header size (W3C) | ~55 bytes |
| Recommended default sample rate | 1–10% (more for errors) |
| Jaeger default retention | 7 days (configurable) |

---

## Real-World Examples

| Company | Tool | Scale |
|---------|------|-------|
| Uber | Jaeger (created it) | Millions of traces/day |
| Netflix | Zipkin → custom | Massive microservices estate |
| Shopify | OpenTelemetry + custom backend | — |
| Most large companies | Datadog APM or Honeycomb | Managed |

---

## Interview Q&A

**Q: A user reports that checkout is slow sometimes. How do you use distributed tracing to diagnose this?**
A: Search Jaeger for traces on the checkout endpoint with P99 latency. Filter to traces above, say, 2 seconds. Open the waterfall — look for the widest spans (where time is spent). Common culprits: a specific downstream service taking 80% of the time, a DB query without an index, or a synchronous call that could be async. Compare a slow trace vs a fast trace side-by-side to spot the difference.

**Q: Why not just use logs for debugging microservices?**
A: Logs tell you what happened inside one service. In a microservices system, a request touches 10 services. Correlating logs across all 10 by timestamp is painful and error-prone. Tracing gives you the causally-linked view automatically — you see the full call tree, durations, and errors in one place. Logs are still essential for detail; tracing gives you the map to know *which* service's logs to look at.

**Q: How does OTel handle context propagation across async boundaries (e.g., a Kafka message)?**
A: OTel propagates trace context through message headers. When publishing to Kafka, the producer injects the current trace context into the message headers. The consumer extracts it and creates a child span that links back to the original trace. This creates a trace that spans the async boundary — you can see the full flow from HTTP request → Kafka publish → consumer processing as one connected trace.