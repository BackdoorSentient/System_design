# 36. Retry & Backoff Strategies

## What is a Retry?

A retry is an automatic re-attempt of a failed operation. Transient failures — brief network blips, temporary overloads, momentary unavailability — are common in distributed systems. Retrying often succeeds without any human intervention.

**The danger:** Naive retries without backoff can **amplify load** on an already-struggling service, making the problem worse. Done wrong, retries are a denial-of-service attack on your own infrastructure.

---

## Q1: What should (and shouldn't) be retried?

### Retry-safe (idempotent operations)
- **GET** requests — reading data doesn't change state
- **DELETE** (if idempotent by design) — deleting something already deleted = no-op
- **PUT** (full resource replacement) — applying same update twice = same result
- Database reads
- Any operation where the server guarantees idempotency (idempotency key pattern)

### Never retry without idempotency
- **POST** without idempotency key — creating a payment twice = double charge
- **Non-idempotent state mutations** — "increment counter" retried = incremented twice
- Operations that already succeeded but whose response was lost (the client timed out but the server committed)

**Idempotency key pattern:** Assign a unique key to each mutation. Server stores the result against the key. On retry, server returns cached result instead of re-executing.
```
POST /payments
Idempotency-Key: uuid-abc-123-def
{"amount": 99.99}

→ Server executes payment, stores result under key abc-123-def
→ Client retries with same key → server returns stored result, no new charge
```

---

## Q2: What are the retry strategies?

### 1. Immediate Retry
Retry instantly after failure. Almost always wrong for transient failures — the service is still processing or still overloaded.

```python
for attempt in range(3):
    try:
        return call_service()
    except Exception:
        pass  # ← retry immediately — don't do this
```

### 2. Fixed Delay
Wait a fixed time between retries.

```python
RETRY_DELAY = 1  # second
for attempt in range(3):
    try:
        return call_service()
    except Exception:
        time.sleep(RETRY_DELAY)
```

Better, but if many clients all retry at exactly the same intervals, they create a **retry wave** — synchronized bursts of traffic at T+1s, T+2s, T+3s.

### 3. Exponential Backoff
Double the wait time after each failure. Gives the service exponentially more time to recover.

```python
base_delay = 1  # second
max_delay = 60  # seconds
max_attempts = 5

for attempt in range(max_attempts):
    try:
        return call_service()
    except Exception:
        if attempt == max_attempts - 1:
            raise
        delay = min(base_delay * (2 ** attempt), max_delay)
        # attempt 0: 1s, attempt 1: 2s, attempt 2: 4s, attempt 3: 8s, attempt 4: 16s
        time.sleep(delay)
```

### 4. Exponential Backoff with Jitter (recommended)
Add randomness to the delay. Prevents the **thundering herd problem** where all clients retry in sync.

```python
import random

delay = min(base_delay * (2 ** attempt), max_delay)
jitter = random.uniform(0, delay)  # Full jitter
time.sleep(jitter)
```

**Full jitter:** `sleep = random(0, min(cap, base * 2^attempt))`

This spreads retries across the window, reducing the peak load on the recovering service.

**AWS recommendation:** Full jitter is the best strategy for distributed systems. It reduces peak retry load by ~50% compared to exponential backoff without jitter.

---

## Q3: What is a retry storm and how do you prevent it?

**Retry storm:** When a service goes down, all clients start retrying simultaneously. Each retry adds load to the recovering service. The recovering service can't start up cleanly because it's immediately overwhelmed by retries.

```
Service B goes down (T=0)
T=1: 1000 clients each retry → 1000 requests hit B (just started recovering)
T=2: 1000 clients retry again → B overwhelmed, crashes again
T=4: 1000 clients retry → same result
→ B can never recover
```

**Prevention:**
1. **Jitter** — spread retries across the time window (as above)
2. **Circuit breaker** — stop retrying after failure threshold; let B recover
3. **Max retry limit** — 3–5 retries maximum, then fail fast
4. **Retry budget** — cap total retries across the system per second (at load balancer or service mesh level)

**Google SRE recommendation:** Each service should only add ~10% overhead through retries. If Service A makes 100 calls to B normally, retries should add at most 10 more calls, not 100.

---

## Q4: What errors should trigger a retry vs immediate failure?

| Error | Retry? | Reason |
|-------|--------|--------|
| `503 Service Unavailable` | ✅ Yes | Server overloaded, transient |
| `429 Too Many Requests` | ✅ Yes (with backoff) | Rate limited, wait and retry |
| `500 Internal Server Error` | ✅ Maybe (idempotent ops) | Might be transient |
| `502 Bad Gateway` | ✅ Yes | Upstream proxy issue, transient |
| `504 Gateway Timeout` | ✅ Yes (idempotent only) | Transient timeout |
| `400 Bad Request` | ❌ No | Your request is malformed |
| `401 Unauthorized` | ❌ No | Auth issue, retrying won't help |
| `403 Forbidden` | ❌ No | Permissions issue |
| `404 Not Found` | ❌ No | Resource doesn't exist |
| `409 Conflict` | ❌ No (usually) | State conflict, needs different handling |
| Network timeout | ✅ Maybe (idempotent only) | Server may have processed request |

---

## Q5: Retry in HTTP clients and libraries

### Python (tenacity library)

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type((ConnectionError, Timeout))
)
def call_payment_service(request):
    return requests.post("/payment", json=request, timeout=5)
```

### Go (standard pattern)

```go
func callWithRetry(ctx context.Context, fn func() error) error {
    backoff := 1 * time.Second
    maxBackoff := 60 * time.Second
    
    for attempt := 0; attempt < 5; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }
        if !isRetryable(err) {
            return err  // fail fast for non-retryable errors
        }
        jitter := time.Duration(rand.Int63n(int64(backoff)))
        time.Sleep(jitter)
        backoff = min(backoff*2, maxBackoff)
    }
    return errors.New("max retries exceeded")
}
```

---

## Q6: Retry at the service mesh level (Istio)

Retries can be configured at the infrastructure level without application code:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  http:
  - retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,retriable-4xx
    timeout: 10s  # total timeout across all retries
```

**Note:** `perTryTimeout × attempts` should be less than total `timeout`. Here: 3 × 2s = 6s < 10s total.

---

## Q7: Deadline propagation

**The hidden retry problem:** Service A retries Service B 3 times. B internally retries Service C 3 times. C internally retries its DB 3 times. A single user request can generate 27 database calls.

**Solution — deadline propagation:**
Pass an absolute deadline (not a timeout) through the entire call chain via a header:

```
Service A: "This request must complete by T+5s"
Service B receives this deadline → if <500ms left, don't bother calling C
Service C receives deadline → if expired, return immediately
```

gRPC supports deadline propagation natively. For HTTP, use a custom header like `X-Request-Deadline`.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Recommended max retry attempts | 3–5 |
| Base delay for exponential backoff | 100ms–1s |
| Max backoff cap | 30–60s |
| Retry overhead budget (Google) | ≤10% of total requests |
| Full jitter reduction in retry peak | ~50% vs no jitter |
| Idempotency key storage TTL | 24 hours (typical) |

---

## Interview Q&A

**Q: Why is exponential backoff with jitter better than exponential backoff alone?**
A: Without jitter, all clients that failed at the same time (e.g., during a brief outage) retry at exactly the same intervals: T+1s, T+2s, T+4s. This creates synchronized bursts that hammer the recovering service at each interval. Jitter adds randomness — each client retries at a slightly different time within the backoff window, spreading the load evenly. AWS benchmarked this and found full jitter reduces collision rates by ~50%, significantly improving recovery speed.

**Q: A payment service times out. Should you retry?**
A: Only if the request is idempotent — i.e., the payment endpoint supports idempotency keys. A timeout means the server may or may not have processed the request (the response was lost, not necessarily the operation). Retrying without an idempotency key risks double-charging. With an idempotency key, the server will return the original result on retry. This is why payment APIs (Stripe, Braintree) require idempotency keys on all POST requests.

**Q: What is the difference between a timeout and a deadline?**
A: A timeout is relative — "wait for 5 seconds from now." A deadline is absolute — "complete by 14:32:05.000Z." Deadlines propagate correctly across service hops: if the original request has a 5-second deadline, every downstream service automatically knows how much time is left. Timeouts don't carry this — Service B doesn't know how much of Service A's timeout has already elapsed. In gRPC, deadlines are first-class citizens. For HTTP, implement via a `X-Request-Deadline` header.