# 24. Distributed Locking

## What is Distributed Locking?

A distributed lock ensures that only **one process/node at a time** performs a critical operation, even when multiple processes run concurrently across different machines.

**Use cases:**
- Only one instance of a cron job runs at a time (distributed cron)
- Preventing double-spend in payment processing
- Leader election in a distributed cluster
- Rate limiting at the resource level (one request at a time per user)
- Inventory reservation during flash sales

---

## Q1: Why can't you use a regular mutex or database row lock?

A regular `threading.Lock()` only works within a single process. A database row lock (`SELECT FOR UPDATE`) works within a single DB but:
- Doesn't prevent multiple processes from acquiring locks on different DB replicas
- Fails if you're trying to lock a resource that spans multiple systems
- The locking service itself can go down, leaving processes blocked

For distributed systems, you need a lock that:
1. Works across machines
2. Has a **TTL (time-to-live)** / lease — auto-expires if the lock holder crashes
3. Is reliable under network partitions

---

## Q2: How does Redis-based distributed locking work?

### Simple Approach: SET NX PX

```
SET lock:resource_id <unique_token> NX PX 30000
```

- `NX` — Only set if Not eXists (atomic: get the lock or fail)
- `PX 30000` — Expire after 30,000ms (30s)
- `<unique_token>` — A random UUID unique to this lock holder

**Acquiring the lock:**
```python
token = str(uuid4())
acquired = redis.set("lock:order_123", token, nx=True, px=30000)
if acquired:
    try:
        do_critical_work()
    finally:
        release_lock("lock:order_123", token)
```

**Releasing the lock (must use Lua script for atomicity):**
```lua
-- Atomic check-and-delete: only release if we own the lock
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

**Why the Lua script?** Without it:
1. Process A reads the lock → still has its token ✓
2. Lock expires (after step 1, before step 2)
3. Process B acquires the lock
4. Process A deletes the lock → deletes Process B's lock! ❌

The Lua script runs atomically on Redis — no race condition.

---

## Q3: What is the Redlock algorithm?

**Problem with single-Redis locking:** Redis itself is a SPOF. If Redis crashes, the lock is gone. If Redis has a replica, there's a window where the primary fails before replication — both processes could acquire the lock.

**Redlock** (by Salvatore Sanfilippo, Redis creator) uses **N independent Redis nodes** (typically 5).

### Algorithm:

1. Get current time T1
2. Try to acquire lock on all N nodes (with a short timeout per node, e.g., 5ms)
3. If acquired on **majority (N/2 + 1) nodes** AND total elapsed time < lock TTL:
   - Lock is acquired
4. If not, release all acquired locks and retry (with jitter)

```
N=5 Redis nodes. Must acquire on 3+.
Lock TTL = 10s.
Must complete all 5 acquisitions in < 10s.
```

**The validity time** = TTL − (T2 − T1) − clock_drift_factor

### Controversy

Martin Kleppmann wrote a detailed critique arguing Redlock doesn't provide strong safety guarantees under certain timing conditions (process pauses, clock drift). He recommends **fencing tokens** for truly safe locking.

### In Practice
For most use cases (cron deduplication, rate limiting, loose coordination), single-Redis locking with TTL is sufficient. Use Redlock only when you need stronger guarantees.

---

## Q4: What are fencing tokens?

**The problem:** A process acquires a lock, gets paused (GC, page fault, OS scheduling), lock TTL expires, another process acquires the lock. Now both processes think they have the lock.

**Fencing tokens** solve this:

1. Lock service issues a **monotonically increasing token** (e.g., 34) when granting a lock
2. Lock holder sends this token with every operation to the protected resource
3. The resource server tracks the highest token seen — rejects any operation with an older token

```
T=0: Process A acquires lock, token=34
T=1: Process A pauses (GC)
T=10: Lock expires. Process B acquires lock, token=35
T=11: Process A resumes, tries to write to storage with token=34
      Storage says: "Seen token 35 already. Reject 34."
T=11: Process B writes to storage with token=35 → accepted
```

Fencing tokens require the storage layer to participate in the protocol. This is a strong guarantee but requires infrastructure support.

---

## Q5: ZooKeeper-based distributed locking

ZooKeeper provides a cleaner locking primitive via **ephemeral sequential znodes**.

### Algorithm (Apache Curator recipe):

1. Each client creates an ephemeral sequential znode: `/locks/lock-0000000001`, `-0000000002`, etc.
2. Client that created the **lowest-numbered** znode holds the lock
3. Other clients watch the znode just below theirs (not all — avoids herd effect)
4. When the lock holder dies, its ephemeral znode disappears → the next client is notified

```
/locks/
  lock-0000000001  ← Process A (lock holder)
  lock-0000000002  ← Process B (watching 001)
  lock-0000000003  ← Process C (watching 002)

A finishes → deletes 001 → B is notified → B now holds lock
```

**Advantages over Redis:**
- ZooKeeper provides linearizable reads — no "acquired from majority" heuristics
- Ephemeral znodes auto-delete on session death (no TTL drift issues)
- No need for Lua scripts

**Disadvantages:**
- ZooKeeper is heavier to operate than Redis
- Higher latency than Redis (consensus on every write)

---

## Q6: When should you use each approach?

| Scenario | Recommendation |
|----------|----------------|
| Simple cron deduplication | Redis SET NX with TTL |
| Leader election in a service | ZooKeeper / etcd (Raft-backed) |
| Inventory reservation (loose) | Redis SET NX |
| Financial transaction safety | ZooKeeper + fencing tokens |
| High throughput locking (1000s/sec) | Redis (lower latency) |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Redis lock acquisition latency | <1ms (single node) |
| Redlock N (typical) | 5 nodes |
| Redlock per-node timeout | 5ms |
| ZooKeeper locking latency | 5–20ms |
| Recommended lock TTL | 10–30s (enough for work + safety margin) |

---

## Interview Q&A

**Q: What happens if the lock holder crashes before releasing the lock?**
A: The lock TTL (expiry) ensures it's automatically released after the set time. This is why TTL is critical — without it, a crashed process would hold the lock forever. The TTL must be long enough for the operation to complete, but short enough to recover quickly from crashes.

**Q: How do you choose the TTL for a distributed lock?**
A: TTL = max expected operation time × safety factor (e.g., 2–3×). For a DB write that takes 100ms, a 5s TTL is reasonable. For a long-running job (30s), use a 90s TTL and implement lock renewal (heartbeat to extend TTL while the job runs).

**Q: Can you use a relational database for distributed locking?**
A: Yes — `SELECT GET_LOCK('resource', 10)` in MySQL or advisory locks in PostgreSQL. It works and has strong consistency, but the database becomes the lock service. It doesn't scale as well as Redis for high lock acquisition rates, and the DB is now a bottleneck for both data operations and coordination.