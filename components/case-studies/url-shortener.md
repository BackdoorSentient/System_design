# Case Study: URL Shortener

---

**Q: What is the scope of a URL shortener system design? What are the functional and non-functional requirements?**

A URL shortener converts a long URL into a short alias (e.g., `https://bit.ly/3xK9mPz`) and redirects users to the original URL when they visit the alias.

**Functional requirements:**
- Shorten a long URL to a short alias (e.g., `short.ly/abc123`)
- Redirect a short URL to the original long URL
- Custom aliases (optional): user specifies the slug
- Expiry: links can have a TTL after which they stop redirecting
- Analytics: count clicks, track referrer, geo, device

**Non-functional requirements:**
- **High availability**: Redirects must work. A broken short link is user-visible. Target 99.99% uptime.
- **Low redirect latency**: Redirects should be < 10ms p99. Users experience this as page load lag.
- **Durability**: A shortened URL must never silently stop working unless explicitly expired.
- **Read-heavy**: Redirects vastly outnumber writes (shortening). Expect 100:1 to 1000:1 read/write ratio.
- **Scale**: Assume 100M URLs shortened total, 10B redirects/day.

---

**Q: How do you estimate scale for a URL shortener?**

**Write (shortening) QPS:**
- 100M total URLs. If the service is 5 years old, that's 100M / (5 × 365 × 86400) ≈ 0.63 URL/sec average.
- Peak: ~10–50 writes/sec. Writes are trivially light.

**Read (redirect) QPS:**
- 10B redirects/day = 10B / 86400 ≈ 115,000 req/sec average.
- Peak (2× average): ~230,000 req/sec.

**Storage:**
- Each URL record: short_code (8 bytes) + long_url (200 bytes avg) + metadata (100 bytes) ≈ 300 bytes.
- 100M records × 300 bytes = 30 GB. Trivially fits on a single DB with room to spare for years.

**Bandwidth (egress):**
- Each redirect response is a 301/302 HTTP response ~200 bytes.
- 115,000 req/sec × 200 bytes = 23 MB/s outbound. Negligible.

**Key insight from estimation:** This is almost entirely a **read problem**. Storage is small, writes are rare, but redirects at 115k req/sec require heavy caching.

---

**Q: How do you generate the short code? Walk through the options and trade-offs.**

The short code is the critical design decision. It must be:
- Short (6–8 characters is standard)
- Unique
- Not guessable (for private links)

**Option 1: Auto-increment ID → Base62 encode**

Assign each URL an auto-incrementing integer ID. Encode the integer in base62 (a-z, A-Z, 0-9).

- 6 chars in base62 = 62^6 = ~56 billion combinations. Plenty.
- ID 1 → `000001`, ID 123456 → `w7h` (short, deterministic)
- Pros: Simple, no collision handling, reversible (decode to get the ID).
- Cons: Sequential — IDs are guessable. `abc123` → `abc124` likely also exists. Bad for private links.

**Option 2: MD5 / SHA-256 hash of the long URL, take first 6–8 chars**

Hash the long URL, take the first 7 base62 characters of the hash.

- Deterministic: same long URL always produces the same short code. Natural deduplication.
- Cons: Hash collisions in the truncated 7-char space. With 100M URLs and 7 base62 chars (62^7 = 3.5T combinations), birthday collision probability at 100M entries ≈ (100M)^2 / (2 × 3.5T) ≈ 0.14%. Need collision resolution: append a counter and rehash.
- Cons: Two users shortening the same URL get the same code — may not be desired if tracking is per-user.

**Option 3: Random token**

Generate a random 7-character base62 string. Check DB for uniqueness before inserting.

- Pros: Unpredictable, no sequential exposure.
- Cons: DB lookup required per generation. Under high write concurrency, may need retries on collision.
- At 100M URLs in a 62^7 = 3.5T space, collision probability per attempt ≈ 100M / 3.5T ≈ 0.003%. Effectively one retry per ~33,000 writes.

**Recommended approach: Auto-increment ID + Base62, with a randomisation layer**

Use a distributed ID generator (Twitter Snowflake-style or a simple database sequence) for uniqueness and non-collision guarantees. Then shuffle the bits of the ID (XOR with a secret mask) before base62-encoding to make the output non-sequential without sacrificing uniqueness.

```
id = next_id()         # e.g., 1000042
obfuscated = id XOR SECRET_MASK
short_code = base62_encode(obfuscated)
```

This gives uniqueness (from the ID), non-guessability (from XOR), and O(1) generation with no collision retries.

---

**Q: What is the database schema for a URL shortener?**

```sql
CREATE TABLE urls (
    id            BIGINT PRIMARY KEY,          -- auto-increment or Snowflake ID
    short_code    VARCHAR(10) UNIQUE NOT NULL, -- 'abc1234'
    long_url      TEXT NOT NULL,               -- original URL, up to 2048 chars
    user_id       BIGINT,                      -- nullable if anonymous
    created_at    TIMESTAMP NOT NULL,
    expires_at    TIMESTAMP,                   -- NULL = never expires
    click_count   BIGINT DEFAULT 0             -- approximate; update async
);

CREATE INDEX idx_short_code ON urls(short_code);
```

The `short_code` index is the only hot read path. Every redirect is a `SELECT long_url WHERE short_code = ?`.

**For analytics (click events)**, use a separate append-only table or stream to Kafka:

```sql
CREATE TABLE clicks (
    id           BIGINT PRIMARY KEY,
    short_code   VARCHAR(10) NOT NULL,
    clicked_at   TIMESTAMP NOT NULL,
    ip_address   INET,
    user_agent   TEXT,
    referrer     TEXT,
    country_code CHAR(2)
);
```

Store clicks asynchronously — never block a redirect on an analytics write.

---

**Q: What is the redirect architecture? What HTTP status code do you use and why does it matter?**

**The redirect flow:**
1. User visits `https://short.ly/abc123`
2. DNS resolves `short.ly` to the service's load balancer
3. Request hits an app server (or CDN edge)
4. App server looks up `abc123` in cache → cache hit → return HTTP redirect
5. Browser follows redirect to the original URL

**301 vs 302:**

| | 301 Permanent | 302 Temporary |
|---|---|---|
| Browser caches it? | Yes — forever | No |
| Analytics captured? | Only on first visit (subsequent hits go direct) | Every visit |
| CDN caches it? | Yes | Depends on headers |
| Use when | URL will never change | Analytics needed, or TTL-based links |

**Most URL shorteners use 302** even though 301 is technically more correct. Reason: with 301, the browser caches the redirect and subsequent clicks bypass your servers entirely — you lose click tracking. Use 301 only for fully permanent, analytics-free shortening.

---

**Q: How do you cache redirects efficiently? What are the cache layers?**

Given 230k req/sec peak, you cannot hit the DB on every redirect. Caching is essential.

**Layer 1: CDN edge (Cloudflare, CloudFront)**
- Short URLs are globally cacheable (same input → same output).
- Cache `GET /abc123 → 302 Location: https://...` at edge nodes.
- TTL: match the link's expiry. For non-expiring links, set TTL = 24h and refresh on access.
- Cache hit ratio goal: 90%+. At 90% hit rate, the origin sees only 23k req/sec instead of 230k.

**Layer 2: Redis (distributed in-process cache)**
- Cache `short_code → long_url` in Redis with TTL matching link expiry.
- Redis GET latency: < 1ms. DB query latency: 5–20ms.
- Memory: 100M records × ~300 bytes = 30 GB. Fits in a single Redis node (use Redis Cluster for HA).
- Eviction: LRU. The hot 10% of links likely handle 90% of traffic — working set fits in 3 GB.

**Layer 3: Application-level in-process cache**
- Small LRU cache per app instance (e.g., 10,000 entries, ~3 MB).
- For extremely hot links (viral URLs), avoids Redis round-trips entirely.
- Cache hit: < 0.1ms.

**Read path with caching:**
```
CDN edge → in-process LRU → Redis → PostgreSQL (rarely hit)
```

---

**Q: How do you handle custom aliases and the uniqueness problem?**

Custom aliases (e.g., `short.ly/black-friday-sale`) are user-specified slugs.

**Uniqueness enforcement:** A `UNIQUE` constraint on `short_code` in the DB is the source of truth. App logic attempts the insert and handles the constraint violation as "alias already taken."

**Race condition:** Two users submitting the same alias simultaneously both check "is it available?" → both see it's free → both try to insert → one succeeds, one gets a unique constraint violation. Handle with:
- DB-level unique constraint (required regardless)
- Application-level retry with a user-facing error message

**Namespace collision:** Custom aliases share the same namespace as auto-generated ones. Reserve a prefix or separate namespace: auto-generated codes are 7 lowercase + digit chars; custom aliases allow mixed case and hyphens. Ensure the encoding of auto-generated IDs never produces a valid custom-alias pattern (or vice versa).

---

**Q: How do you handle expired links?**

**On read:** Check `expires_at` on every cache hit. If the link is expired, return HTTP 410 Gone (not 404 — 410 signals intentional permanent removal; browsers and crawlers treat it differently).

**On cache:** Store `expires_at` alongside the URL in cache. Set the cache TTL to `expires_at - now()` so the entry naturally evaporates.

**Background cleanup (soft delete):** A periodic job (cron or Celery task) runs daily and DELETEs rows where `expires_at < NOW()`. This keeps the DB small and indexes tight. Do not hard-delete immediately on read — it adds latency to the hot path.

**Reclaiming short codes:** If codes are numeric/ID-based, expired codes can be reused. If codes are hashes, retired codes should be tombstoned in a bloom filter to prevent reuse — a user might have bookmarked the old link and should get 410, not a new destination.

---

**Q: How do you scale the write path when generating IDs across multiple app servers?**

With multiple app servers, you cannot use a single DB sequence safely without coordination overhead. Options:

**Option 1: Database auto-increment (simplest)**
- One table with `AUTO_INCREMENT`. Each insert gets a unique ID.
- Works fine at low-to-moderate write rates (< 1,000 writes/sec).
- Bottleneck: the DB sequence becomes a serialisation point at extreme scale.

**Option 2: Pre-allocated ID ranges**
- Each app server fetches a block of 1,000 IDs from a central counter and uses them locally.
- After exhausting the block, fetch the next. Network round-trips reduced by 1000×.

**Option 3: Twitter Snowflake IDs**
- 64-bit ID: 41 bits timestamp (ms) + 10 bits machine ID + 12 bits sequence.
- Each machine generates up to 4,096 unique IDs per millisecond with no coordination.
- IDs are time-ordered (useful for pagination), unique across machines, and fit in a BIGINT.

**Option 4: UUID v4**
- Random 128-bit ID. Zero coordination needed.
- Con: 128-bit is wider than BIGINT; random insertion causes B-tree index fragmentation.
- Not recommended for primary keys in high-write databases.

**Recommended:** Snowflake IDs for any system at serious scale.

---

**Q: What does the complete system architecture look like?**

```
User → Cloudflare CDN → Load Balancer (AWS ALB)
                              ↓
                      Redirect Service (stateless, 10 pods)
                         ↙         ↘
                    Redis Cluster    PostgreSQL (primary + 2 read replicas)
                                          ↑
                      Shortener Service (low-traffic write path)
                              ↓
                         Snowflake ID generator
                              ↓
                         Kafka (click events)
                              ↓
                      Analytics Service → ClickHouse (OLAP)
```

**Key design decisions summarised:**
- CDN caches 90%+ of redirects at the edge — origin sees ~10% of traffic.
- Redis caches `short_code → long_url` for the remaining 10%.
- PostgreSQL primary handles writes; read replicas handle cache misses.
- Click events are fire-and-forget to Kafka; never block the redirect.
- Snowflake IDs ensure unique, non-guessable codes across all write pods.
- `expires_at` checked on cache read; background job cleans DB nightly.

---

**Q: What are the failure modes and how do you handle them?**

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Redis down | All redirects hit DB | DB read replicas handle load; add DB connection pooling (PgBouncer). Circuit breaker to avoid hammering a recovering DB |
| DB primary down | Writes fail; reads continue via replicas | Auto-failover (RDS Multi-AZ promotes replica in ~60s). Queue writes in Redis during failover |
| App pod crash | Some in-flight requests fail | Load balancer health checks remove the pod in < 30s; stateless pods are replaced by autoscaling |
| Hot link (viral) | One short code gets millions of concurrent reads | In-process LRU catches this; CDN edge also absorbs spike |
| Hash collision | Two long URLs get same short code | Unique DB constraint enforced; application retries with modified input |
| Code enumeration | Attacker iterates all codes to find private links | Use XOR obfuscation on sequential IDs, or full random tokens. Rate-limit redirect endpoint per IP |