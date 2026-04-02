# 29. Quorum

## What is a Quorum?

A quorum is the **minimum number of nodes** that must agree for an operation to be considered valid. It's the mechanism that enables distributed systems to maintain consistency without requiring all nodes to be available.

**Core insight:** If any write quorum and any read quorum overlap, reads always see at least one copy of the latest write.

---

## Q1: How does quorum work mathematically?

With N replicas, W write replicas, R read replicas:

**Quorum condition:** W + R > N

This ensures the write set and read set always share at least one node.

```
N = 5, W = 3, R = 3:
W + R = 6 > 5 ✓

Any group of 3 nodes (write set) and any group of 3 nodes (read set)
must overlap by at least 1 node (since 3 + 3 - 5 = 1).
That overlapping node has the latest write.
```

**If W + R ≤ N:** Reads might miss the latest write (no guaranteed overlap).

---

## Q2: What are the common W/R configurations?

| Config | W | R | Description |
|--------|---|---|-------------|
| **Strong consistency** | N | 1 | Write to all, read from one. Low read latency, high write latency |
| **Quorum (balanced)** | N/2+1 | N/2+1 | Standard tradeoff |
| **Read-heavy** | N | 1 | Fast reads, slow writes |
| **Write-heavy** | 1 | N | Fast writes, slow reads |
| **Eventual consistency** | 1 | 1 | No overlap guarantee; very fast but stale reads possible |

**Cassandra consistency levels** map to these:

| Cassandra Level | Meaning |
|-----------------|---------|
| `ONE` | W=1 or R=1 |
| `QUORUM` | W or R = N/2 + 1 (across all DCs) |
| `LOCAL_QUORUM` | Quorum within local DC only |
| `EACH_QUORUM` | Quorum in each DC |
| `ALL` | All replicas must respond |

---

## Q3: How does quorum work in DynamoDB?

DynamoDB (described in the Dynamo paper) uses **N=3 replicas** by default:

- **W=2, R=2** (quorum)
  - 2 + 2 = 4 > 3 ✓ → strong consistency
  - Any 2-node write set overlaps any 2-node read set by 1 node

**Tunable consistency:**
- `ConsistentRead: true` → R=2 (quorum, guaranteed latest)
- `ConsistentRead: false` → R=1 (eventual, might be stale, lower latency, cheaper)

DynamoDB stores data in 3 AZs. A quorum read/write survives 1 AZ failure (can still reach 2 of 3 replicas).

---

## Q4: What is sloppy quorum?

**Strict quorum:** The W and R nodes must be from the designated N replicas for that key.

**Sloppy quorum:** If some of the N designated nodes are unreachable, write to other available nodes temporarily (hinted handoff).

**Example (N=3, W=2):**
Normal nodes for key X: A, B, C
C is down. Write goes to A, B (strict quorum met).
But A is also slow. Write goes to B and D (not in X's shard).
D stores a "hint" that it's holding data for C. When C recovers, D forwards the hint.

**Sloppy quorum guarantees:**
- High availability (writes succeed even when designated nodes are down)
- **Does NOT guarantee strong consistency** — during failure, R=2 might not overlap with W=2 if writes went to non-standard nodes

**Used by:** Amazon DynamoDB (sloppy quorum enabled by default for availability)

---

## Q5: Quorum in consensus algorithms

Raft and Paxos both use quorum for their operations:

**Raft:**
- Election: Candidate needs majority votes (N/2 + 1)
- Log commit: Leader needs majority acknowledgment (N/2 + 1)

**Why majority?** Any two majorities of N nodes share at least one member. That shared node guarantees the new leader has seen all committed entries.

```
N = 5. Two majorities of 3 each:
Group 1: {A, B, C}
Group 2: {C, D, E}
Overlap: {C}
```

---

## Q6: Quorum reads and the stale read problem

Even with W + R > N, stale reads can occur in certain scenarios:

**Scenario 1: Read-repair race**
1. Write goes to A, B (W=2, N=3, C not yet updated)
2. Client reads from B, C (R=2)
3. B has new value, C has old value
4. Client takes the value with the highest timestamp → gets new value ✓

But if the system uses **last-write-wins** and timestamps are not perfectly synchronized (clock skew), the "latest" timestamp might actually be the older value.

**Scenario 2: Non-overlapping partitions**
During a network partition, if sloppy quorum is used, it's possible to write to non-designated nodes and later read from the designated nodes (which haven't received the update yet).

**Mitigations:**
1. **Strict quorum** (no sloppy quorum) — guarantees overlap
2. **Vector clocks / version vectors** — detect concurrent writes instead of relying on timestamps
3. **Read repair** — on read, write back the latest value to lagging replicas

---

## Q7: Quorum for multi-DC deployments

In multi-datacenter setups, a standard quorum might require cross-DC coordination on every operation (adding latency).

**`LOCAL_QUORUM` in Cassandra:**
- Write to quorum within local DC only (e.g., 2 of 3 local replicas)
- Asynchronously replicate to remote DCs
- Reads also use local quorum

Trade-off: Faster operations, but a DC-level failure might serve stale reads from the surviving DC.

**`EACH_QUORUM`:**
- Requires quorum in EVERY datacenter
- Strongest cross-DC consistency, highest latency

---

## Numbers to Remember

| Config | N | W | R | Notes |
|--------|---|---|---|-------|
| DynamoDB strong | 3 | 2 | 2 | Default quorum |
| DynamoDB eventual | 3 | 2 | 1 | Faster reads |
| Cassandra QUORUM (N=3) | 3 | 2 | 2 | 4 > 3 ✓ |
| Cassandra ALL | 3 | 3 | 3 | Strongest, slowest |
| Raft cluster (3 nodes) | 3 | 2 | — | Majority |

---

## Interview Q&A

**Q: You have N=5, W=3, R=2. Is this a valid quorum?**
A: W + R = 5, which is NOT > N=5 (it's equal to). This does not guarantee overlap. You need W + R > N. Set R=3 for strong consistency (3+3=6 > 5). R=2 here would be eventual consistency.

**Q: Why does DynamoDB offer eventual consistency at all? Why not always use quorum?**
A: For read-heavy workloads, R=1 (eventual consistency) is faster, costs less (you only read 1 replica), and for use cases like shopping cart or recommendation reads, a slightly stale value is perfectly acceptable. DynamoDB charges less for eventually consistent reads. The application developer chooses the right trade-off for each query.

**Q: What happens to quorum during a node failure (N=3, W=2)?**
A: If one node fails, N_available=2. W=2 means you still need 2 nodes for writes — barely achievable (both remaining nodes must respond). If another node fails (N_available=1), you can't achieve W=2 and writes fail. This is the trade-off of higher W — better consistency, less fault tolerance. With W=1, you can write as long as any single node is alive (maximum availability, no consistency guarantee).