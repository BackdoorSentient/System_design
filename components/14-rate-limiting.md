# 07_rate_limiting.md — Rate Limiting

---

**Q: Why is rate limiting essential in production systems? What problems does it solve?**

Rate limiting controls the number of requests a client can make to a service within a given time window. Without it:

- **DoS/DDoS protection**: A single client (or botnet) can flood your service with millions of requests, exhausting CPU, memory, DB connections, and bandwidth.
- **Fair usage**: Prevent one tenant in a multi-tenant system from consuming all resources at the expense of others.
- **API monetization**: Enforce subscription tiers (Free: 100 req/day, Pro: 10k req/day).
- **Cost control**: Prevent runaway scripts from causing unexpected cloud compute and API costs.
- **Backend protection**: Shield downstream services (DBs, third-party APIs) from being overwhelmed.
- **Abuse prevention**: Limit brute-force login attempts, credential stuffing, scraping.

**What to rate limit on**:
- Per IP address (coarse, breaks behind NAT).
- Per API key / user ID (accurate, requires authentication).
- Per endpoint (some endpoints are more expensive than others — limit `/search` more strictly than `/ping`).
- Per tenant in multi-tenant SaaS.

---

**Q: Explain the Token Bucket algorithm. What are its properties and trade-offs?**

**Concept**: Imagine a bucket with a maximum capacity of N tokens. Tokens are added at a fixed rate (R tokens per second). Each request consumes one token (or more for "expensive" operations). If the bucket has tokens, the request is allowed. If empty, the request is rejected or queued.

**Properties**:
- **Allows bursting**: If traffic is below the rate for a while, tokens accumulate (up to bucket capacity). A burst of requests can consume all accumulated tokens immediately. Example: capacity=100 tokens, rate=10/sec. After 10 seconds of silence, 100 tokens are saved. A burst of 100 requests is allowed instantly.
- **Average rate is enforced**: Over any long time window, throughput cannot exceed R req/sec.
- **Smooth vs bursty**: Burst capacity is the key tunable. Small capacity = smooth traffic. Large capacity = allows significant bursts.

**Implementation**:
```python
class TokenBucket:
    def __init__(self, capacity, rate):
        self.capacity = capacity
        self.tokens = capacity
        self.rate = rate           # tokens per second
        self.last_refill = now()

    def allow(self):
        elapsed = now() - self.last_refill
        self.tokens = min(self.capacity,
                          self.tokens + elapsed * self.rate)
        self.last_refill = now()
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

**Distributed implementation (Redis)**:
- Store `{tokens, last_refill_time}` per user in Redis.
- Use a Lua script to make the read-modify-write atomic (avoids race conditions).
- On each request: refill based on elapsed time, check if token available, decrement.

**Trade-offs**:
- Pro: Natural burst handling. Simple to reason about.
- Con: Requires precise time tracking. In distributed systems, slight timing discrepancies can cause over- or under-limiting.

---

**Q: Explain the Leaky Bucket algorithm. How does it differ from token bucket?**

**Concept**: Imagine a bucket with a hole in the bottom. Requests pour in from the top (at any rate). Water (requests) leaks out the bottom at a fixed, constant rate. If the bucket is full, new requests overflow (are rejected).

**Properties**:
- **Smooths traffic to a constant rate**: Output is always at the leak rate, regardless of how bursty the input is. No bursts downstream — the rate is perfectly uniform.
- **Queue-based**: The bucket acts as a FIFO queue. Requests are processed one by one at the fixed rate.
- **Overflow rejection**: Requests arriving when the queue is full are rejected immediately.

**Key difference from token bucket**:
- Token bucket: Allows bursts up to bucket capacity. Downstream service sees variable request rate (up to burst capacity).
- Leaky bucket: Output is always at constant rate. Downstream service sees perfectly smooth, even traffic. Better for protecting downstream services that need predictable load.

**When to use each**:
- **Token bucket**: API rate limiting where you want to allow short bursts (human users who might click rapidly, client retry logic).
- **Leaky bucket**: Shaping traffic to a backend that needs smooth load (a slow upstream payment processor, rate-limited third-party API you're proxying).

---

**Q: What is the Fixed Window Counter algorithm? What is its critical flaw?**

**Concept**: Divide time into fixed windows (e.g., 1-minute buckets). Count requests per user per window. Reject requests when the count exceeds the limit.

```
Window: 12:00:00–12:00:59 → user A: 3 requests (limit: 5 → OK)
Window: 12:01:00–12:01:59 → user A: 4 requests (limit: 5 → OK)
```

**Implementation**: Redis `INCR key` + `EXPIRE key 60`. Simple and fast.

**The critical flaw — boundary burst attack**:
A user makes 5 requests at 12:00:55 (end of window 1) and 5 more at 12:01:00 (start of window 2). Both windows are within their limit of 5. But in the 2-second span of 12:00:55–12:01:00, you've received 10 requests — double the intended rate. The fixed window boundary creates a 2x burst vulnerability.

---

**Q: What is the Sliding Window Counter and how does it fix the fixed window problem?**

**Sliding Window Log (exact)**:
Store a timestamp for each request in a sorted set. For each new request:
1. Remove all timestamps older than `now - window_size` from the set.
2. Count remaining entries.
3. If count < limit, add the current timestamp and allow. Else reject.

- Exact but memory-intensive: O(N) storage per user where N is the request limit.
- 1,000 users × 1,000 requests limit = 1M entries in memory.

**Sliding Window Counter (approximation, production-preferred)**:
Uses two fixed window counters (current and previous) with a weighted formula to approximate the sliding window.

```
sliding_count = prev_window_count × (1 - elapsed_fraction) + current_window_count

Where elapsed_fraction = (current_time - current_window_start) / window_size
```

Example:
- 1-minute window limit: 100 requests.
- Previous window had 80 requests. Current window (20% elapsed) has 15 requests.
- Sliding estimate: 80 × 0.8 + 15 = 79. Under limit, allow.

**Why this works**: The previous window's requests are weighted by how much of the window still overlaps with the current sliding window. As time passes, the previous window's weight decreases.

**Error**: Maximum error is ~0.003% for uniform traffic. Excellent approximation for all practical purposes. Used by Cloudflare and Kong.

**Storage**: Only 2 counters per user per window. O(1) per user. Redis-friendly with `INCR` + `EXPIRE`.

---

**Q: How do you implement distributed rate limiting across multiple API gateway instances?**

**Problem**: You have 10 load-balanced API gateway nodes. Each tracks rate limits locally. A user sends 100 requests, 10 to each node. Each node sees only 10 requests and allows all of them. Effective limit is 10× the intended limit.

**Solutions**:

**1. Centralized Redis counter (most common)**:
- All API gateway nodes increment the same key in a shared Redis instance.
- Redis `INCR` is atomic. No race conditions.
- Lua script for read-check-increment atomicity.
- **Latency**: +0.5–2ms per request (Redis round-trip).
- **Redis SPOF**: Use Redis Sentinel or Redis Cluster for HA.

```lua
-- Atomic Lua script in Redis
local count = redis.call('INCR', KEYS[1])
if count == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[1])  -- set TTL on first increment
end
return count
```

**2. Redis Cell (GCRA — Generic Cell Rate Algorithm)**:
- Redis module `redis-cell` implements token bucket natively in Redis with a single atomic command: `CL.THROTTLE user_id 100 10 1` (bucket size 100, 10 tokens/sec, consume 1).
- Returns: allowed/denied, remaining tokens, retry-after seconds. Very clean.

**3. Approximate local rate limiting with sync**:
- Each node tracks counts locally. Periodically (every 100ms) sync to Redis.
- Lower Redis traffic but allows small overage between syncs.
- Good for non-strict rate limits where slight overage is acceptable.

**4. Rate limit at the API Gateway layer**:
- Kong, AWS API Gateway, Envoy, Nginx Plus all have built-in rate limiting with Redis backend.
- Don't implement from scratch if a managed solution exists.

---

**Q: What should you return when rate limiting a request? How do you help clients retry correctly?**

**HTTP Status Code**: `429 Too Many Requests`.

**Standard headers** (align with IETF RFC 6585 and common conventions):
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30                        # seconds until the client can retry
X-RateLimit-Limit: 100                 # total allowed requests in window
X-RateLimit-Remaining: 0              # remaining requests in current window
X-RateLimit-Reset: 1704067200         # Unix timestamp when window resets
X-RateLimit-Window: 60                # window size in seconds
```

**Response body**:
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "You have exceeded 100 requests per minute.",
    "retry_after": 30
  }
}
```

**Client behavior**:
- Respect `Retry-After`. Do not immediately retry — this worsens the problem.
- Implement exponential backoff with jitter for retries: `wait = base_delay * 2^attempt + random(0, 1)`.
- Surface rate limit headers to allow clients to self-throttle proactively.

**Graceful degradation**: Rate-limited responses should still be fast (served from the rate limiter layer, not reaching the application). A 429 should respond in <1ms.

---

**Q: What is the difference between rate limiting and throttling? How does throttling protect services?**

**Rate limiting**: Hard enforcement. Requests exceeding the limit are rejected with 429.

**Throttling**: Soft enforcement. Requests exceeding the threshold are slowed down (queued, delayed) rather than rejected. The system processes them at a controlled pace.

**Throttling as backpressure**:
- When a service is overwhelmed, it responds slowly rather than crashing. The slowdown propagates upstream as backpressure.
- A service returning responses in 10 seconds instead of 100ms signals to the caller to slow down.
- Circuit breakers (Hystrix, resilience4j) detect this pattern and short-circuit calls to a degraded service.

**Adaptive rate limiting**:
- Adjust rate limits dynamically based on system load.
- If CPU > 80%, reduce the rate limit by 50%.
- If error rate > 5%, reduce rate limit.
- More sophisticated than fixed limits but requires more machinery.
