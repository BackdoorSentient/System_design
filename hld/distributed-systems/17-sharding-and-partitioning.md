# 17. Sharding & Partitioning

## What is Sharding?

Sharding (horizontal partitioning) splits a dataset across multiple machines — each machine holds a subset of rows. Unlike replication (same data on multiple nodes), sharding means each node has *different* data.

**Why shard?**
- A single machine can't hold all the data (storage limit)
- A single machine can't handle all the writes (throughput limit)
- Read/write latency increases as tables grow

---

## Q1: What are the three main sharding strategies?

### 1. Hash-Based Sharding

Apply a hash function to the shard key, then mod by number of shards:

```
shard = hash(user_id) % N
```

**Example:** user_id = 12345 → hash = 987654 → 987654 % 4 = shard 2

**Pros:**
- Uniform distribution (if hash is good and keys are random)
- Simple to implement

**Cons:**
- **Resharding is expensive** — adding a shard changes N, so nearly all keys remap to new shards. Must migrate most data.
- Cannot do range queries efficiently (keys are scattered)
- Hot key problem if one key has disproportionate traffic

**Used by:** Early MongoDB, many custom sharding implementations

---

### 2. Range-Based Sharding

Split keyspace into contiguous ranges. Each shard owns a range.

```
Shard 1: user_id 1 – 1,000,000
Shard 2: user_id 1,000,001 – 2,000,000
Shard 3: user_id 2,000,001 – 3,000,000
```

**Pros:**
- Range queries are efficient (all data in a range is co-located)
- Easy to add new shard at the end for sequential keys

**Cons:**
- **Hotspots** — if keys are sequential (auto-increment IDs, timestamps), all new writes go to the last shard. The "hot shard" problem.
- Uneven data distribution if data isn't uniformly distributed across ranges

**Used by:** HBase, BigTable, Cassandra (within a partition)

---

### 3. Directory-Based Sharding

Maintain a **lookup table** (the directory) that maps each key (or key range) to a shard.

```
lookup_table["user_123"] → shard_5
lookup_table["user_456"] → shard_2
```

**Pros:**
- Flexible — can relocate any key to any shard without changing the algorithm
- Easy to handle hotspots by moving specific keys

**Cons:**
- The lookup table itself is a **single point of failure**
- Extra network hop (must query directory before routing)
- Directory becomes a bottleneck at scale

**Used by:** Custom sharding in some large systems; Vitess (MySQL sharding) uses a topo server

---

## Q2: What is the shard key and why does it matter?

The **shard key** is the column(s) used to determine which shard a row belongs to. Choosing it wrong is one of the most expensive mistakes in distributed system design.

**Good shard key properties:**
1. **High cardinality** — many unique values (user_id: millions; country_code: bad, only ~200)
2. **Uniform distribution** — values spread evenly to avoid hot shards
3. **Aligns with access patterns** — queries should hit as few shards as possible (avoid scatter-gather)
4. **Immutable** — changing the shard key requires moving the row to another shard

**Anti-patterns:**
- **Timestamp as shard key** → all writes go to "latest" shard
- **Status field as shard key** → 90% of rows are `status=active`, one shard overloaded
- **Low-cardinality key** → impossible to balance load

---

## Q3: What is a hotspot and how do you fix it?

A **hotspot** (hot shard) is a shard receiving significantly more reads/writes than others.

**Causes:**
- Celebrity accounts (one user_id has 100M followers — writes to their shard)
- Sequential keys (all new rows go to the last range shard)
- Seasonal traffic (all Black Friday orders have today's timestamp)

**Solutions:**

1. **Key salting** — append a random suffix to the key before hashing
   ```
   shard = hash(user_id + "_" + random(0,10)) % N
   ```
   Spreads one key across 10 shards. Reads must query all 10 and merge. Write-heavy hot keys benefit most.

2. **Dedicated shard for hot keys** — identify top-K hot keys and route them to a dedicated shard with extra resources.

3. **Application-level caching** — cache hot keys in Redis so DB shard isn't hit for every read.

4. **Consistent hashing with virtual nodes** — (see topic 18) reduces hotspot during node additions.

---

## Q4: What is cross-shard (scatter-gather) query and why is it expensive?

A query that doesn't include the shard key must be sent to **all shards** and results merged.

```sql
SELECT * FROM orders WHERE product_id = 42;
-- If sharded by user_id, product_id=42 could be on ANY shard
-- Must query all N shards → N network round trips → merge results
```

**Cost:** O(N) network calls, O(N) compute, results arrive at different times.

**Mitigations:**
- Design schema so common queries include shard key
- Maintain a secondary index shard (inverted index mapping product_id → list of user_ids)
- Denormalize data across multiple shard keys

---

## Q5: How does resharding work and why is it painful?

**The problem:** You start with 4 shards but need 8 as data grows. With hash-based sharding (hash % N), changing N remaps most keys.

```
Old: hash(key) % 4 → shard 2
New: hash(key) % 8 → shard 6  (different!)
```

Virtually every key needs to move. This means:
- Moving terabytes of data
- Reads/writes must be served correctly during the migration (dual reads or downtime)

**Solutions:**
1. **Consistent hashing** — only ~1/N fraction of keys move when a node is added (see topic 18)
2. **Pre-split** — start with more shards than you need (e.g., 1024 virtual shards on 8 physical nodes). To scale, just move virtual shards to new nodes.
3. **Online resharding** — tools like Vitess (MySQL), MongoDB's built-in sharding balancer do this live.

---

## Q6: What is the difference between vertical and horizontal partitioning?

| | Horizontal Partitioning (Sharding) | Vertical Partitioning |
|---|---|---|
| Split by | Rows | Columns |
| Each partition has | Subset of rows, all columns | All rows, subset of columns |
| Goal | Scale data volume / write throughput | Separate hot columns from cold columns |
| Example | Users 1-1M on shard A, 1M-2M on shard B | user_id/name/email in one table, large bio/avatar in another |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| MongoDB default chunk size | 128 MB |
| Cassandra default virtual nodes per physical node | 256 |
| Practical shards before scatter-gather becomes a problem | >20-50 |
| Rule of thumb: start resharding at | 80% of shard capacity |

---

## Real-World Examples

| System | Strategy | Shard Key |
|--------|-----------|-----------|
| Instagram (early) | Hash-based | user_id |
| Cassandra | Consistent hash | Partition key |
| MongoDB | Range or hash | User-defined |
| Vitess (YouTube) | Range | user_id / order_id |
| DynamoDB | Hash | Partition key |
| Twitter | User-based | user_id (timeline shard) |

---

## Interview Q&A

**Q: How would you shard a users table for a social network with 1 billion users?**
A: Use user_id as the shard key with consistent hashing. Choose a high-cardinality key (UUID or auto-increment). Pre-provision virtual shards (e.g., 1024 virtual, 16 physical). Avoid timestamp or geographic sharding — they create hotspots. Most user queries include user_id so scatter-gather is rare.

**Q: How do you handle joins across shards?**
A: Avoid them by design. Options: (1) Denormalize — store user name alongside each post (redundancy is okay). (2) Application-side join — query each shard separately and join in application layer. (3) Rethink schema — if you frequently join A and B, co-locate them on the same shard key.

**Q: What's wrong with using `created_at` as a shard key for a logging system?**
A: All new log entries arrive with today's timestamp, so all writes go to the most recent time-range shard. This creates a permanent hot shard for the latest partition. Better: hash on a composite key like `(service_id, minute_bucket)` to distribute load while still allowing time-range queries per service.