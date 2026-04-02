# 35. Bulkhead Pattern

## What is the Bulkhead Pattern?

Named after the watertight compartments in a ship's hull — if one compartment is breached, the others remain sealed and the ship stays afloat. In software, the bulkhead pattern **isolates components** so that a failure or overload in one part doesn't consume resources needed by other parts.

**The problem it solves:** Without bulkheads, all services in an application share the same resource pools (thread pool, connection pool, memory). If one slow downstream service holds threads, it can exhaust the shared pool and bring down the entire application — even parts that have nothing to do with that service.

---

## Q1: How does resource pool exhaustion cause failures?

**Scenario without bulkheads:**

```
Application has a shared thread pool: 200 threads

Normal traffic:
  Order Service calls: 50 threads
  Payment Service calls: 50 threads
  Notification Service calls: 50 threads
  Free: 50 threads

Payment Service becomes slow (30s timeouts):
  Order Service: 50 threads (normal)
  Payment Service: 200 threads (all held waiting for timeouts!)
  Notification Service: 0 threads (starved)
  Order Service: 0 threads (starved)

→ Entire application is unresponsive, not just payment
```

**With bulkheads:**
```
Separate thread pools:
  Order pool: 50 threads
  Payment pool: 50 threads
  Notification pool: 50 threads

Payment becomes slow:
  Payment pool: 50 threads (all held — payment calls fail)
  Order pool: 50 threads (fully operational)
  Notification pool: 50 threads (fully operational)

→ Only payment-related calls fail, rest of app works fine
```

---

## Q2: What are the two types of bulkheads?

### 1. Thread Pool Isolation

Each downstream dependency gets its **own dedicated thread pool**.

```java
// Hystrix / Resilience4j thread pool bulkhead
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(25)           // max 25 concurrent calls to this service
    .maxWaitDuration(Duration.ZERO)   // fail immediately if pool is full
    .build();

Bulkhead paymentBulkhead = Bulkhead.of("payment-service", config);
Bulkhead inventoryBulkhead = Bulkhead.of("inventory-service", config);
```

**Pros:**
- Complete isolation — one slow service can't affect others
- Can assign more threads to critical services, fewer to non-critical

**Cons:**
- Thread context switching overhead
- Many threads idle if traffic is uneven
- Total thread count = sum of all pools (can be large)

---

### 2. Semaphore Isolation

Instead of separate thread pools, limit **concurrent in-flight calls** using semaphores. Runs in the caller's thread.

```java
// Resilience4j semaphore bulkhead
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(20)           // max 20 concurrent calls
    .maxWaitDuration(Duration.ofMs(10)) // wait up to 10ms for a slot
    .build();
```

**Pros:**
- Much lighter than thread pool (no extra threads)
- Less overhead for reactive/non-blocking code

**Cons:**
- No timeout protection — if a call blocks its thread, the caller's thread is stuck
- Less isolation than thread pool (still uses caller threads)

**When to use which:**
- Thread pool: blocking I/O calls, need true isolation, can afford the threads
- Semaphore: non-blocking/async code, many downstream services, want low overhead

---

## Q3: How does the bulkhead relate to connection pool sizing?

Database connection pools are a classic bulkhead application.

**The HikariCP (most common Java DB pool) rule of thumb:**
```
connections = (core_count × 2) + effective_spindle_count
```

For an 8-core machine: `8 × 2 + 1 = 17 connections` is often optimal — more connections hurt performance due to contention.

**Multi-tenant bulkhead (connection pool per tenant):**
```
Tenant A pool: 20 connections (paid tier)
Tenant B pool: 5 connections (free tier)
Default pool: 10 connections
```

A free-tier user flooding the system can't starve paid users.

**Per-service connection pools:**
```
Payment service DB pool: 30 connections
Analytics service DB pool: 10 connections
Reporting service DB pool: 5 connections
```

Analytics queries (slow, analytical) can't hold connections that payment service needs.

---

## Q4: Bulkhead at the infrastructure level — Kubernetes

In Kubernetes, bulkheads manifest as resource limits and namespace isolation.

**Resource quotas (namespace-level bulkhead):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payment-quota
  namespace: payment-service
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

**Pod resource limits (pod-level bulkhead):**
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

A payment pod with a memory leak can't OOM-kill the notification pod if they have separate limits.

**Node affinity / node pools:**
Critical services (payment, auth) on dedicated node pools — a noisy neighbor workload on shared nodes can't affect them.

---

## Q5: Bulkhead at the API gateway level

Rate limiting per client/tenant is a bulkhead at the entry point:

```
Client A (free tier): 100 req/min
Client B (pro tier): 10,000 req/min
Client C (enterprise): 100,000 req/min
```

A misbehaving or DDoSing free-tier client can't starve enterprise clients.

Similarly, separate API gateways or rate limit pools per product area:
```
/api/payment/* → payment rate limiter (strict, financial)
/api/search/*  → search rate limiter (generous, read-only)
/api/upload/*  → upload rate limiter (bandwidth-based)
```

---

## Q6: Bulkhead + Circuit Breaker together

These patterns are **complementary** and should be used together:

```
Incoming Request
      │
      ▼
[Bulkhead] ──── pool full? ──► BulkheadFullException → fallback
      │
      ▼
[Circuit Breaker] ── OPEN? ──► CircuitBreakerOpenException → fallback
      │
      ▼
[Retry with backoff]
      │
      ▼
[Downstream Service]
```

- **Bulkhead** limits concurrent calls and resource usage
- **Circuit Breaker** detects failure rates and stops calling a broken service
- **Retry** handles transient failures
- All three together = resilient service-to-service communication

---

## Q7: Trade-offs

| Aspect | Thread Pool Bulkhead | Semaphore Bulkhead |
|--------|---------------------|-------------------|
| Isolation | Strong (separate threads) | Weak (caller threads) |
| Overhead | High (thread context switch) | Low |
| Timeout support | Yes (thread can be interrupted) | No (caller thread blocked) |
| Async support | Difficult | Natural |
| Suitable for | Blocking I/O, critical services | Async, many services |

**The cost of over-bulkheading:** Too many thread pools = hundreds of idle threads consuming memory. Each thread = ~512KB–1MB stack. 200 threads = ~200MB minimum. Balance isolation with resource efficiency.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Java thread stack size | 512KB–1MB |
| Typical thread pool per service (Hystrix) | 10 threads default |
| HikariCP recommended pool size | (cores × 2) + 1 |
| Max semaphore permits (typical) | 20–50 per downstream |
| OOM risk threshold | ~1000+ threads in one JVM |

---

## Interview Q&A

**Q: How is the bulkhead pattern different from the circuit breaker?**
A: They solve different problems and work at different levels. The bulkhead limits how many concurrent calls can be made to a dependency — it prevents resource exhaustion in the caller. The circuit breaker monitors failure rates and stops calling a service that's clearly broken — it prevents cascading failures and gives the downstream time to recover. The bulkhead prevents the caller from being overwhelmed; the circuit breaker prevents the callee from being overwhelmed. Use both together.

**Q: You have 50 microservices. Do you create a separate thread pool bulkhead for every dependency of every service?**
A: Only for the high-risk ones — dependencies that are slow, unreliable, or whose failure would cascade badly. Group less critical dependencies into shared pools. Prioritize: (1) external third-party APIs (always unpredictable), (2) slow database or analytics queries, (3) non-critical internal services. For fast, reliable internal services on the same LAN, the overhead of a bulkhead may not be worth it. Apply bulkheads where you have data showing latency or reliability issues.

**Q: What happens when a bulkhead is full?**
A: The request is immediately rejected with a `BulkheadFullException` (fail fast). The caller handles this with a fallback — return cached data, enqueue for retry, return a degraded response. The key is that the rejection happens immediately (no waiting), so the caller's thread is released. This is the point — fast failure keeps the caller healthy even when the downstream is overwhelmed.