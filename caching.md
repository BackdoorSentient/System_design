# 02_caching.md — Caching

---

**Q: Why is caching so important in system design, and what are the primary caching layers?**

Caching stores copies of expensive-to-compute or expensive-to-fetch data in faster storage so subsequent requests are served cheaply.

**Why it matters**:
- DB queries: 10–100ms. Redis cache hit: 0.1–1ms. That's a 10–1000x latency improvement.
- Reduces database load — a DB that handles 10k req/s without cache might need to handle only 500 req/s with 95% cache hit rate.
- Enables horizontal read scaling without scaling the DB.

**Caching layers in a web stack**:
1. **Client-side**: Browser caches HTTP responses via `Cache-Control`, `ETag`, `Last-Modified`.
2. **CDN**: Edge nodes cache static assets and even API responses globally.
3. **Reverse proxy / API Gateway**: Nginx, Varnish can cache responses at the gateway level.
4. **Application-level**: In-process cache (Java `HashMap`, Python `functools.lru_cache`). Fastest, but local to one instance.
5. **Distributed cache**: Redis, Memcached. Shared across all app instances.
6. **Database query cache**: MySQL's query cache (deprecated in 8.0), PostgreSQL shared_buffers.

---

**Q: Compare Redis and Memcached. When do you choose one over the other?**

| Dimension | Redis | Memcached |
|---|---|---|
| Data structures | Strings, Hashes, Lists, Sets, Sorted Sets, Streams, Bitmaps, HyperLogLog | Strings only (blobs) |
| Persistence | RDB snapshots + AOF write-ahead log | None (purely in-memory) |
| Replication | Primary-replica, Redis Sentinel, Redis Cluster | None natively |
| Clustering | Redis Cluster (hash slots, 16384 slots) | Client-side sharding only |
| Transactions | MULTI/EXEC, Lua scripts | None |
| Pub/Sub | Yes | No |
| Memory efficiency | Slightly higher overhead | Slightly more memory-efficient for plain strings |
| Multi-threading | Single-threaded event loop (I/O threads added in 6.0) | Multi-threaded |
| Latency | ~0.1–0.5ms | ~0.1–0.3ms |

**Choose Memcached when**:
- You only need simple key-value string caching.
- You need maximum raw throughput for a simple caching use case.
- Multi-threaded performance matters and you're CPU-bound.

**Choose Redis when** (almost always):
- You need rich data structures (sorted sets for leaderboards, lists for queues).
- You need persistence/durability.
- You need pub/sub messaging.
- You need atomic operations (INCR, SETNX for distributed locks).
- You need Lua scripting for complex atomic operations.
- You need clustering with automatic resharding.

**In practice**: Redis has essentially won. Most new architectures default to Redis.

---

**Q: Explain cache-aside (lazy loading), write-through, write-behind, and read-through patterns.**

**Cache-aside (Lazy Loading)**
The application is responsible for the cache. On a miss, the app fetches from DB and populates the cache.

```
read(key):
  value = cache.get(key)
  if value is None:
    value = db.query(key)
    cache.set(key, value, ttl=300)
  return value

write(key, value):
  db.write(key, value)
  cache.delete(key)   # invalidate, not update
```

- Pros: Only requested data is cached (no cold-start waste). Cache failures are non-fatal.
- Cons: First request after a miss is slow (cache stampede risk). Data can be stale until TTL expires.
- Used by: Most web applications. Default pattern.

**Write-through**
Every write goes to both cache and DB synchronously.

```
write(key, value):
  cache.set(key, value)
  db.write(key, value)
```

- Pros: Cache is always consistent with DB.
- Cons: Every write incurs DB latency (no write performance benefit). Cache may fill with data never read.

**Write-behind (Write-back)**
Writes go to cache immediately; DB writes are async/batched.

```
write(key, value):
  cache.set(key, value)
  queue.enqueue(DBWrite(key, value))  # async
```

- Pros: Very fast writes. Good for write-heavy workloads (analytics counters, metrics aggregation).
- Cons: Risk of data loss if cache dies before the async write completes. Complex to implement.
- Used by: Gaming leaderboards, real-time analytics counters.

**Read-through**
Cache sits in front of DB and manages its own population on misses. The app only talks to the cache.

- Similar to cache-aside but the cache itself fetches from DB on miss, not the application.
- Pros: Simpler application code.
- Cons: Cache becomes a more complex component with DB knowledge.

---

**Q: What is the cache stampede (thundering herd) problem and how do you solve it?**

**The problem**: A popular cache key expires. At that exact moment, thousands of requests simultaneously get a cache miss, all query the DB simultaneously, and all try to repopulate the cache. The DB is overwhelmed.

**Solutions**:

1. **Mutex/lock**: Only one request fetches from DB; others wait. Use Redis `SETNX` to acquire a lock. Others poll the cache until it's populated. Risk: lock holder dies → all waiters hang.

2. **Probabilistic early expiration (XFetch)**: Before the TTL actually expires, probabilistically recompute the cache slightly early. The closer to expiration, the higher the probability of early refresh. Prevents the simultaneous expiry cliff.

3. **Background refresh**: A background job refreshes cache keys before they expire. Application always gets cached data; the refresh is not triggered by a user request.

4. **Stale-while-revalidate**: Return the stale value immediately and trigger an async refresh. Common in HTTP caching (`Cache-Control: stale-while-revalidate=60`).

5. **Jitter on TTL**: Instead of `ttl=3600`, use `ttl = 3600 + random(0, 300)`. Prevents all keys set at the same time from expiring simultaneously.

---

**Q: Explain LRU vs LFU eviction policies. When does each fail?**

When a cache is full and a new item needs to be inserted, an existing item must be evicted.

**LRU — Least Recently Used**
Evicts the item that was accessed least recently.

- Assumption: Recently accessed items are likely to be accessed again (temporal locality).
- Implemented with: A doubly-linked list + hashmap. O(1) get and put. Redis uses an approximation (samples 5–10 keys randomly and evicts the oldest among them) to save memory.
- Fails when: Scanning a large dataset (cache pollution). A sequential scan evicts all hot items because each scan item is "most recent" for a moment.

**LFU — Least Frequently Used**
Evicts the item accessed least frequently.

- Assumption: Frequently accessed items are important regardless of recency.
- Implemented with: A frequency min-heap or frequency buckets.
- Fails when: Items popular in the past but no longer needed stay in cache (frequency history is stale). New items that will be very popular get evicted before they accumulate frequency.
- Fix: **LFU with decay** — reduce frequency counts over time so old popularity fades.

**Other policies**:
- **FIFO**: Evict oldest inserted. Simple but ignores access patterns.
- **Random**: Evict random item. Surprisingly competitive in practice.
- **ARC (Adaptive Replacement Cache)**: Combines recency and frequency. Self-tuning. Used in ZFS, some DBs.
- **W-TinyLFU**: Used in Caffeine (Java) and Redis 4.0 `allkeys-lfu`. Frequency filter + LRU segments. Best hit rate in most workloads.

**Redis eviction policies** (set via `maxmemory-policy`):
- `noeviction` — return error when full.
- `allkeys-lru` — LRU across all keys.
- `volatile-lru` — LRU only among keys with TTL set.
- `allkeys-lfu` — LFU across all keys.
- `allkeys-random` — random eviction.

---

**Q: What is a CDN and how does CDN caching work? Explain cache invalidation strategies.**

A CDN (Content Delivery Network) is a geographically distributed network of edge servers (PoPs — Points of Presence) that cache content close to users.

**How it works**:
1. User requests `https://cdn.example.com/image.jpg`.
2. DNS resolves to the nearest CDN edge node.
3. If the edge has a cached copy (cache hit), it returns immediately (latency ~5–20ms vs ~200ms to origin).
4. If not (cache miss), the edge fetches from origin, caches it, and returns it.

**Cache control headers**:
- `Cache-Control: public, max-age=86400` — cache for 24 hours, public (CDN can cache).
- `Cache-Control: private` — only browser can cache, not CDN.
- `Cache-Control: s-maxage=3600` — CDN-specific TTL (overrides `max-age` for shared caches).
- `Vary: Accept-Encoding` — different cached copies for gzip vs brotli vs uncompressed.

**Cache invalidation strategies**:

1. **TTL expiry**: Let items expire naturally. Simple, but stale data for TTL duration.
2. **Versioned URLs**: `image.v3.jpg` — change the filename/hash on every deploy. Old URL is still cached; new URL misses and fetches fresh. Preferred for static assets.
3. **Purge API**: CloudFront, Fastly, Cloudflare all provide APIs to explicitly invalidate specific URLs or patterns. Can be triggered on deploy. First N purge requests are free; beyond that, it can be expensive.
4. **Surrogate-Control / Cache-Tags**: Tag responses with logical tags (`product-123`). On a product update, purge all responses tagged `product-123`. Fastly and Cloudflare support this.

**Numbers**:
- CDN cache hit rate of 90%+ for static assets is typical.
- Cloudflare has ~300 PoPs; Akamai has ~4,000.
- Edge-to-user latency: <20ms for most users in served regions.

---

**Q: What is cache consistency, and how do you handle it in distributed systems?**

Cache consistency is the challenge of keeping the cache and the source of truth (DB) in sync when data changes.

**Strong consistency**: Every read reflects the latest write. Impossible to achieve in a distributed cache without making every read go to the DB (defeating the purpose).

**Eventual consistency**: After a write, the cache will eventually reflect the new value (after TTL expires or explicit invalidation). Acceptable for most applications.

**Common approaches**:

1. **Cache invalidation on write**: When the DB is written, delete (not update) the cache key. The next read will repopulate. Deletion is safer than update — avoids race conditions where an older value overwrites a newer one.
   - Race condition risk: Two concurrent writes can interleave: write A → write B → delete B → delete A. The cache is now empty but DB has B's value. Next read gets B. This is fine.
   - Worse race: Read misses → DB query → write happens → old value written to cache. Now cache has stale data until TTL. Mitigate with short TTLs or leases.

2. **CDC (Change Data Capture) + cache invalidation**: Use Debezium to stream DB changes (binlog) into a Kafka topic. A consumer reads the topic and invalidates/updates the cache. Decouples the app from cache management.

3. **Read-your-writes consistency**: After a user writes data, route their subsequent reads to the DB directly (not cache) for a short window. Ensures they see their own writes.

---

**Q: What is the difference between horizontal and vertical cache scaling?**

**Vertical scaling**: Give the cache server more RAM. Simple, but has a physical limit and creates a SPOF.

**Horizontal scaling** (distributed cache):
- **Client-side sharding**: The application uses consistent hashing to deterministically assign keys to cache nodes. Adding/removing nodes causes ~1/N of keys to be remapped (minimal disruption). Libraries: Twemproxy, Ketama.
- **Redis Cluster**: Built-in sharding. Keyspace divided into 16,384 hash slots. Each node owns a range of slots. Cluster can automatically reshards. Supports up to ~1,000 nodes.

**Consistent hashing**: Key is hashed to a position on a ring. Each node owns a segment of the ring. When a node is added, only the keys in its segment are remapped (~1/N keys). Virtual nodes (vnodes) improve distribution.

**Replication for read scaling**: Redis primary-replica. Reads go to replicas; writes go to primary. Replicas are eventually consistent with the primary (replication lag typically <1ms on LAN).

---

**Q: What are common caching anti-patterns to avoid?**

1. **Caching mutable shared state without invalidation**: Set TTL but never explicitly invalidate. Users see stale data for the full TTL window.

2. **Caching everything**: Cache stores data that's never read again, evicting actually hot data. Be selective — cache high-read, low-write data.

3. **Using cache as primary data store**: If the cache is lost, data is gone. Cache should be a copy, not the source of truth (unless write-behind with guaranteed durability).

4. **Not handling cache failures**: If Redis goes down, app should degrade gracefully by querying the DB, not fail completely. Use circuit breakers around cache calls.

5. **Large values in cache**: Storing 10MB objects in Redis blocks the event loop for other operations (Redis is single-threaded). Break large objects into smaller keys.

6. **Hot key problem**: A single cache key is accessed millions of times per second (celebrity tweet, viral product). Single Redis node becomes a bottleneck. Solutions: local in-process cache for that key, key replication across multiple Redis nodes, or read replicas.
