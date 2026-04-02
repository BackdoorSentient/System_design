# 16. Replication

## What is Replication?

Replication means keeping a copy of the same data on multiple machines (replicas). The primary goals are:
- **High availability** — serve reads/writes even if one node fails
- **Read scalability** — spread read load across replicas
- **Geo-distribution** — serve users from a nearby replica

---

## Q1: What are the three main replication strategies?

### Leader-Follower (Master-Slave)
One node is the **leader** (master) — all writes go here. The leader sends a replication log to **followers** (slaves/replicas), which apply the same writes. Reads can go to followers.

```
Client → [Leader] ──write log──► [Follower 1]
                   ──write log──► [Follower 2]
Client (read) ──────────────────► [Follower 1]
```

**Used by:** MySQL, PostgreSQL, MongoDB, Redis

**Trade-offs:**
| Pro | Con |
|-----|-----|
| Simple to reason about | Single write bottleneck |
| Reads scale horizontally | Replication lag on followers |
| Failover is straightforward | Followers serve stale reads |

**Replication lag** — followers are typically milliseconds to seconds behind the leader. Under heavy write load or slow network, lag can grow to minutes.

---

### Multi-Leader (Master-Master)
Multiple nodes can accept writes. Each leader replicates to the others.

**Used by:** MySQL Cluster, CockroachDB (in some topologies), Google Docs (local device = a "leader")

**Use case:** Multi-datacenter deployments where writes need to continue even during a WAN partition.

**The big problem — write conflicts:**
- User A updates row X on DC1 leader
- User B updates row X on DC2 leader (simultaneously)
- When both try to replicate to each other → **conflict**

**Conflict resolution strategies:**
1. **Last-write-wins (LWW)** — highest timestamp wins. Fast but loses data.
2. **Application-level merge** — app defines merge logic (e.g., Google Docs CRDTs)
3. **Custom conflict handlers** — `ON CONFLICT DO` hooks

**Trade-offs:**
| Pro | Con |
|-----|-----|
| No single write bottleneck | Write conflict complexity |
| Survive datacenter failure | Hard to implement correctly |
| Lower write latency per DC | Risk of diverging replicas |

---

### Leaderless (Dynamo-Style)
No leader — any replica can accept writes. Uses **quorum** to achieve consistency.

**Write:** Send to all N replicas, wait for W acknowledgments.
**Read:** Send to all N replicas, wait for R responses, take latest version.

**Condition for strong consistency:** W + R > N

**Example (N=3, W=2, R=2):** 2+2 > 3 ✓ — at least one node in a read set overlaps with a write set.

**Used by:** Amazon DynamoDB, Apache Cassandra, Riak

**Handling stale replicas — two mechanisms:**
1. **Read repair** — client reads from multiple nodes, detects stale value, writes newest back to lagging node.
2. **Anti-entropy** — background process compares replicas and syncs differences (using Merkle trees).

**Trade-offs:**
| Pro | Con |
|-----|-----|
| No SPOF (single point of failure) | No linearizability by default |
| Tunable consistency (tweak W/R) | Concurrent write conflicts |
| High availability, AP system | More complex client logic |

---

## Q2: How does replication lag cause problems?

### Three consistency guarantees violated by lag:

**1. Read-your-writes (monotonic reads)**
- User submits a post, reads their profile page from a lagging follower → post not visible yet
- Fix: Route user's own reads to the leader for a short TTL after their write

**2. Monotonic reads**
- User reads at time T1 (from follower A) → sees record
- Same user reads at time T2 (from follower B with more lag) → record gone
- Fix: Sticky session — pin user to the same replica for reads

**3. Consistent prefix reads**
- In a sharded DB, different shards replicate at different speeds
- Reads can see events out of causal order
- Fix: Causal consistency tokens or single shard for causally related data

---

## Q3: What is synchronous vs asynchronous replication?

| | Synchronous | Asynchronous |
|---|---|---|
| Leader waits for? | Follower ACK before confirming write to client | Does not wait |
| Durability | High (no data loss on failover) | Data loss possible |
| Latency | Higher | Lower |
| Availability | Follower down → leader blocks | Leader works independently |
| Used by | Semi-sync in MySQL, 1 sync replica in Postgres | Most read replicas |

**Semi-synchronous (common in production):** One follower is sync, rest are async. Good balance.

---

## Q4: How does failover work in leader-follower replication?

1. **Detect failure** — leader stops responding to heartbeats for N seconds (e.g., 30s timeout)
2. **Elect new leader** — follower with most up-to-date replication log is promoted
3. **Reconfigure clients** — DNS update or load balancer redirects writes to new leader
4. **Old leader rejoins** — must accept new leader, not try to split-brain

**Potential problems:**
- **Split-brain** — old leader comes back, thinks it's still leader. Now two nodes accepting writes. Fix: STONITH (Shoot The Other Node In The Head) — fencing to ensure old leader is dead.
- **Lost writes** — async replicas on new leader may be behind old leader's last ACK'd writes. Must discard or recover these.

---

## Q5: How does Postgres logical vs physical replication differ?

| | Physical (WAL streaming) | Logical |
|---|---|---|
| Unit | Raw disk blocks | SQL operations |
| Cross-version | No (same major version) | Yes |
| Selective tables | No (whole cluster) | Yes |
| Use case | Standby/failover | ETL, selective sync |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Typical async replication lag | 1–100ms on LAN |
| Acceptable lag before user impact | ~1 second |
| MySQL semi-sync timeout | 10s default (falls back to async) |
| Cassandra default replication factor | 3 |
| Quorum for N=3 | W=2, R=2 |

---

## Real-World Examples

| System | Strategy | Notes |
|--------|-----------|-------|
| MySQL / PostgreSQL | Leader-follower | Sync optional, async default |
| MongoDB | Replica set (leader-follower) | Auto-failover via Raft election |
| Cassandra | Leaderless | Tunable consistency (ONE, QUORUM, ALL) |
| DynamoDB | Leaderless | 3 replicas across AZs |
| Redis Cluster | Leader-follower per shard | Async replication |

---

## Interview Q&A

**Q: When would you choose multi-leader over single-leader?**
A: When you need writes to survive a datacenter outage or want to reduce write latency in geo-distributed systems (users write to local DC). The tradeoff is conflict resolution complexity — I'd only use it if cross-DC write availability is a hard requirement.

**Q: A user says "I just posted a comment but I can't see it." What's happening?**
A: Classic replication lag. The write went to the leader but the read is being served by a lagging follower. Fix: Route reads-after-own-writes to the leader for that user, or use sticky sessions on a low-lag replica.

**Q: How does Cassandra handle concurrent writes to the same key on different nodes?**
A: Last-write-wins based on client-supplied timestamp. This can silently drop data if clocks aren't synchronized. For CRDTs or counter use cases, Cassandra has special column types. In practice, avoid concurrent writes to the same key if correctness matters.