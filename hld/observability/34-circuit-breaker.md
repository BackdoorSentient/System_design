# 34. Circuit Breaker Pattern

## What is the Circuit Breaker Pattern?

Named after the electrical circuit breaker, this pattern **stops calling a failing service** when it's clearly broken, allowing it time to recover — rather than hammering it with requests that will fail anyway.

**The problem it solves:** In a microservices system, if Service B is slow or down, Service A keeps sending requests that timeout after 30 seconds each. Service A's thread pool fills up waiting for timeouts → Service A becomes slow → Service C (which calls A) starts timing out → cascading failure. This is called a **cascade** or **timeout storm**.

The circuit breaker detects that B is failing, **opens the circuit**, and immediately rejects new calls to B (fast-fail) until B recovers.

---

## Q1: What are the three states of a circuit breaker?

```
                  failure threshold exceeded
    CLOSED ─────────────────────────────────► OPEN
      ▲                                         │
      │                                         │ wait (reset timeout)
      │                                         ▼
      └──────────── HALF-OPEN ◄────────────────┘
           success          failure
           (close)          (reopen)
```

### CLOSED (normal operation)
- All requests pass through to the downstream service
- Circuit breaker counts failures
- If failures exceed threshold (e.g., 50% of last 20 calls fail), → transition to OPEN

### OPEN (tripped)
- All requests are **immediately rejected** with a fallback (no actual call to downstream)
- Error is returned instantly (fail fast) — no waiting for timeout
- After a configurable **reset timeout** (e.g., 60 seconds), → transition to HALF-OPEN

### HALF-OPEN (recovery probe)
- Allows a **limited number of test requests** through to the downstream service
- If they succeed → close the circuit (service has recovered)
- If they fail → reopen the circuit (service still broken, wait again)

---

## Q2: What are the configurable parameters?

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| **Failure threshold** | % of failures to trip | 50% |
| **Minimum calls** | Calls before evaluating threshold | 10–20 |
| **Sliding window** | Evaluate last N calls or last N seconds | 20 calls or 60s |
| **Reset timeout** | How long to wait before HALF-OPEN | 30–60s |
| **Half-open max calls** | Test requests before deciding | 3–5 |
| **Slow call threshold** | Duration beyond which a call counts as "slow" | 2s |
| **Slow call rate threshold** | % of slow calls to trip | 100% |

---

## Q3: How does a circuit breaker prevent cascading failures?

Without circuit breaker:
```
User Request → Service A → Service B (30s timeout, B is down)
                            ← TimeoutException (after 30s)
              Thread held for 30s × 100 concurrent users = thread pool exhausted
              Service A now slow/unresponsive
User Request → Service C → Service A (now slow) → cascade continues
```

With circuit breaker (after it trips):
```
User Request → Service A → Circuit Breaker (OPEN)
                            ← CallNotPermittedException (immediately, <1ms)
              Thread released immediately
              Service A remains healthy
              B gets no load while recovering
```

The circuit breaker preserves the health of the **caller** by failing fast, and gives the **callee** time to recover without being overwhelmed.

---

## Q4: What is a fallback?

A fallback is the response returned when the circuit is OPEN (or when a call fails). It should degrade gracefully — providing partial functionality rather than a hard error.

**Fallback strategies:**

| Strategy | Example |
|----------|---------|
| **Return cached data** | Return last-known user profile (slightly stale) |
| **Return default value** | Return empty recommendations list instead of error |
| **Return static response** | Return "service temporarily unavailable" page |
| **Try alternative service** | Failover to a secondary endpoint or read replica |
| **Queue for later** | Accept the request, process when service recovers |

```python
@circuit_breaker(failure_rate_threshold=50, reset_timeout_seconds=60)
def get_recommendations(user_id):
    return recommendation_service.get(user_id)

def get_recommendations_with_fallback(user_id):
    try:
        return get_recommendations(user_id)
    except CircuitBreakerOpenException:
        return get_cached_recommendations(user_id)  # fallback
    except Exception:
        return []  # default empty fallback
```

---

## Q5: Resilience4j vs Hystrix

**Hystrix** (Netflix, now in maintenance mode) was the pioneer. **Resilience4j** is the modern replacement.

| | Hystrix | Resilience4j |
|---|---|---|
| Status | Maintenance mode (2018) | Actively maintained |
| Language | Java | Java (functional, no external deps) |
| Thread isolation | Thread pool per command | Semaphore-based (lighter) |
| Reactive support | Limited | Full (RxJava, Reactor) |
| Metrics | Hystrix Dashboard | Micrometer → Prometheus/Grafana |
| Configuration | Archaius (complex) | Simple functional API |

**Resilience4j example (Java):**
```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)          // 50% failure rate trips circuit
    .waitDurationInOpenState(60s)      // wait 60s before HALF-OPEN
    .slidingWindowSize(20)             // evaluate last 20 calls
    .minimumNumberOfCalls(10)          // need at least 10 calls
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

CircuitBreaker cb = CircuitBreaker.of("payment-service", config);

Supplier<PaymentResult> decorated = CircuitBreaker
    .decorateSupplier(cb, () -> paymentService.charge(request));

Try<PaymentResult> result = Try.ofSupplier(decorated)
    .recover(CallNotPermittedException.class, ex -> fallbackResponse());
```

---

## Q6: Circuit breaker in service meshes

In Istio/Envoy, circuit breaking is configured at the infrastructure level — no application code changes:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5      # trip after 5 consecutive 5xx errors
      interval: 30s                     # evaluation window
      baseEjectionTime: 30s            # eject for 30s minimum
      maxEjectionPercent: 50           # eject at most 50% of instances
```

This ejects individual **pod instances** (not the entire service) from the load balancer when they're failing.

---

## Q7: Circuit Breaker vs Retry — when to use which?

| | Retry | Circuit Breaker |
|---|---|---|
| Best for | Transient failures (network blip, brief overload) | Sustained failures (service down, overloaded) |
| Risk | Retry storm (amplifies load on struggling service) | Misses short recovery windows |
| Combine? | Yes — circuit breaker wraps the retry | Retry inside, circuit breaker outside |

**Combined pattern:**
```
Call with retry (3 attempts, exponential backoff)
    → wrapped by circuit breaker
    → if circuit opens, stop retrying immediately
```

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Failure threshold (typical) | 50% |
| Minimum calls before evaluation | 10–20 |
| Reset timeout | 30–60s |
| Hystrix default thread pool size | 10 per command |
| Fast-fail latency (circuit OPEN) | <1ms |
| Typical timeout without CB | 30s × concurrent = thread pool exhaustion |

---

## Interview Q&A

**Q: Why does the circuit breaker have a HALF-OPEN state instead of going directly from OPEN to CLOSED?**
A: Going directly from OPEN to CLOSED after a timeout would flood the recovering service immediately with full traffic — potentially re-tripping it. HALF-OPEN sends a small probe (3–5 test requests) first. Only if those succeed does it return to CLOSED and restore full traffic. This gives the downstream service a chance to stabilize before receiving its full load.

**Q: A circuit breaker trips open. What should users experience?**
A: A graceful degradation — not an error page. Good fallbacks: serve stale cached data, return an empty list (e.g., no recommendations), show a "feature temporarily unavailable" message while keeping the rest of the page functional. The user should be able to continue their core task. Never let the circuit breaker exception bubble up as a raw 500 error.

**Q: How do you tune circuit breaker thresholds?**
A: Start with production data. Look at your P99 latency baseline and set the slow-call threshold above it (e.g., if P99=200ms, set slow threshold=1s). Set failure threshold based on your SLO — if 99.9% success rate is required, 0.1% natural error rate is normal, so set threshold at 10%+ to avoid false trips. Use a sliding window large enough to avoid noise (20 calls or 60 seconds). Monitor circuit state transitions — too many trips = threshold too sensitive; never trips during incidents = too lenient.