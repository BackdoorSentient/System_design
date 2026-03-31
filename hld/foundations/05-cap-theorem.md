# CAP Theorem

---

**Q: What is the CAP theorem, and what do the three letters stand for?**

The CAP theorem, proved by Eric Brewer in 2000, states that a distributed data store can
guarantee at most two of the following three properties simultaneously:

- **Consistency (C):** Every read receives the most recent write or an error. All nodes see
  the same data at the same time. This is linearizability — the system behaves as if there
  is a single, up-to-date copy of the data.

- **Availability (A):** Every request receives a response (not an error), though the response
  may not contain the most recent write. The system stays operational and responsive.

- **Partition Tolerance (P):** The system continues to operate even when network messages
  between nodes are dropped or delayed — i.e., the network is split into isolated groups.

**The key insight:** In any real distributed system, network partitions *will* happen —
hardware fails, switches drop packets, data centers lose connectivity. Therefore, partition
tolerance is not optional. You must design for P. This reduces the real choice to:

> **During a partition, do you sacrifice Consistency (stay available) or Availability
> (stay consistent)?**

This is why CAP is really a CP vs AP trade-off.

---

**Q: Why is Partition Tolerance non-negotiable in practice?**

A network partition is any scenario where two or more nodes cannot communicate. This happens
routinely:

- A NIC fails on a node.
- A switch drops packets between two racks.
- A data center loses its upstream link for 30 seconds.
- A cloud region's internal network degrades.

If you choose not to tolerate partitions (CA), your system must halt entirely any time two
nodes lose connectivity — because proceeding risks diverging state. This means any network
blip takes down your database. That's not a production-grade system.

In practice: every distributed database tolerates partitions. The question is only what
behaviour you guarantee *during* that partition.

---

**Q: What does a CP system do during a partition, and which databases choose this?**

A **CP system** (Consistent + Partition Tolerant) prioritises correctness over availability.
During a network partition, rather than risk serving stale or conflicting data, the system
**refuses requests** (returns an error or blocks) until the partition heals and quorum can
be re-established.

**Behaviour during partition:**
- Writes are rejected if the system cannot confirm the write reached a quorum of nodes.
- Reads are rejected (or blocked) if the system cannot confirm it has seen the latest write.
- The system is unavailable for the isolated minority partition.

**Real-world CP systems:**

| System | Why CP |
|--------|--------|
| **ZooKeeper** | Returns errors during partition; leader election requires quorum |
| **etcd** | Raft consensus — minority nodes refuse reads/writes |
| **HBase** | Relies on HDFS and ZooKeeper; stops on quorum loss |
| **Google Spanner** | TrueTime + Paxos; globally consistent, sacrifices availability |
| **PostgreSQL (sync replication)** | With `synchronous_commit=on`, write blocks until replica confirms |
| **MongoDB (w:majority)** | With majority write concern, rejects writes if quorum unreachable |

**When to choose CP:**
- Financial transactions — a bank cannot show two clients different account balances.
- Inventory systems — overselling one item to two buyers is worse than a brief outage.
- Leader election, distributed locks — correctness is paramount.
- Configuration management (ZooKeeper/etcd) — all nodes must agree on config.

---

**Q: What does an AP system do during a partition, and which databases choose this?**

An **AP system** (Available + Partition Tolerant) prioritises uptime over consistency.
During a network partition, nodes on both sides of the split continue to accept reads and
writes, even though they cannot sync with each other. When the partition heals, the system
reconciles the diverged state through conflict resolution.

**Behaviour during partition:**
- Both sides of the partition accept reads and writes independently.
- The same key may be written differently on each side.
- After healing, the system merges diverged writes — using last-write-wins, vector clocks,
  or application-level logic.
- Clients may read stale data (eventual consistency).

**Real-world AP systems:**

| System | Why AP |
|--------|--------|
| **Cassandra** | All nodes accept writes; tunable consistency (default: eventual) |
| **DynamoDB** | Eventual consistency by default; strong consistency optional but slower |
| **CouchDB** | Multi-master replication; merge conflicts with MVCC |
| **Riak** | Vector clocks for conflict resolution |
| **Amazon S3** | Strong consistency added in 2020, but historically AP |
| **DNS** | Changes propagate over minutes/hours; always responds with cached data |

**When to choose AP:**
- Social media feeds — showing a post 500ms late is fine; going down is not.
- Shopping carts — Amazon famously chose AP; seeing a stale cart is better than an error.
- Metrics, analytics, logging — approximate counts beat unavailability.
- Geo-distributed systems — cross-region consistency latency is too high for many workloads.

---

**Q: Walk me through a concrete partition scenario for a CP vs AP system.**

**Setup:** Three-node cluster: N1 (primary), N2, N3. A network partition splits the cluster
into {N1} and {N2, N3}. A client writes key `balance = 500` to N1.

**CP system (e.g., etcd with Raft):**
- N1 cannot reach a quorum (needs 2 of 3 nodes).
- N1 rejects the write and returns an error to the client.
- N2 and N3 also reject writes (they are a quorum but have no elected leader).
- No stale reads are possible. System is unavailable until partition heals.
- After healing: N1 re-joins, leader re-elected, normal operation resumes.

**AP system (e.g., Cassandra with consistency=ONE):**
- The client writes `balance = 500` to N1. N1 accepts and acknowledges immediately.
- Meanwhile, a second client writes `balance = 300` to N2.
- Both writes succeed. The cluster now has diverged state: N1 says 500, N2/N3 say 300.
- Reads during partition may return either value depending on which node is hit.
- After healing: last-write-wins (based on timestamp) — the later write survives. The other
  is discarded. Data loss is possible.

The CP system protected data integrity at the cost of returning errors.
The AP system stayed responsive at the cost of data divergence.

---

**Q: What is the PACELC theorem, and why does it extend CAP?**

CAP only describes behaviour *during* a network partition. PACELC, proposed by Daniel
Abadi in 2012, extends this to cover the trade-off that exists *even when the network is
healthy*:

```
If Partition (P):  choose between Availability (A) and Consistency (C)
Else (E):          choose between Latency (L) and Consistency (C)
```

**Why PACELC matters:** Even without a partition, a distributed system must choose: do you
wait for all replicas to confirm a write (consistent but slower), or do you acknowledge
after the first node and sync replicas asynchronously (lower latency but momentarily stale)?

**PACELC classifications of real systems:**

| System | Partition behaviour | Normal behaviour | Label |
|--------|--------------------|--------------------|-------|
| DynamoDB | AP | EL (low latency, eventual) | PA/EL |
| Cassandra | AP | EL (tunable, default eventual) | PA/EL |
| MongoDB | CP | EC (majority write = consistent) | PC/EC |
| Google Spanner | CP | EC (globally consistent) | PC/EC |
| MySQL (async replica) | PA | EL | PA/EL |
| MySQL (sync replica) | PC | EC | PC/EC |
| Zookeeper / etcd | CP | EC | PC/EC |

**The practical takeaway:** Most high-traffic systems (social feeds, recommendation engines,
analytics) are PA/EL — they favour low latency and availability everywhere. Financial systems
and coordination services are PC/EC — they pay the latency cost for correctness everywhere.

---

**Q: Is CAP a binary choice? Can a system offer "tunable" consistency?**

Yes — several databases expose consistency as a per-request knob, letting you make the
CP vs AP trade-off at query time rather than system design time.

**Cassandra consistency levels** (per-operation):

| Level | Meaning | Trade-off |
|-------|---------|-----------|
| `ONE` | 1 replica responds | Fastest, most available, stale reads possible |
| `QUORUM` | Majority of replicas respond | Balanced — strong consistency achievable |
| `ALL` | All replicas respond | Strongest, but any replica failure = error |
| `LOCAL_QUORUM` | Quorum within local DC | Low cross-region latency, local consistency |

**The quorum trick for strong consistency:**
With replication factor N=3, QUORUM = 2. If you use QUORUM writes *and* QUORUM reads, then
write quorum + read quorum = 4 > 3 = N. At least one node is always in both the write and
read quorum. That node has the latest write. Therefore, QUORUM reads always return the most
recent write — strong consistency, without using `ALL`.

**DynamoDB:** `ConsistentRead=true` on a read pays ~2x latency but guarantees you see the
latest write. Default reads are eventually consistent (hit any replica, including stale ones).

**Implication for design:** Don't label your whole system "CP" or "AP". Identify which
operations need which guarantees. A social media app might use eventual consistency for
timeline reads (PA/EL — fast, tolerate stale) but strong consistency for payment writes
(PC/EC — correct, tolerate latency).

---

**Q: How does Google Spanner achieve global strong consistency, apparently defeating CAP?**

Spanner is often cited as "CA" — globally consistent and always available. This is misleading.
Spanner is CP. It tolerates partitions by becoming unavailable for the partitioned minority.
What Spanner does differently is make partitions *extremely rare and short* — and provide
very high availability as a result.

**How Spanner achieves this:**

1. **TrueTime API:** Google's global atomic clock infrastructure (GPS + atomic clocks in each
   data centre). TrueTime exposes time as an interval `[earliest, latest]` with a bounded
   uncertainty of ~7ms. Spanner uses this to assign globally-ordered commit timestamps — no
   two transactions get the same timestamp.

2. **Paxos replication groups:** Each shard is replicated via Paxos across 5 replicas in
   3+ geographic regions. Writes require a quorum (3 of 5).

3. **External consistency:** A transaction that commits after another transaction began is
   guaranteed to have a higher timestamp. Reads are served at a specific timestamp, so
   Spanner can serve strongly-consistent reads from any replica — it just waits for the
   TrueTime uncertainty window (up to ~14ms) to pass before serving the read.

**The result:** Spanner achieves ~99.999% availability (5 nines) in practice. It does this
not by defeating CAP but by engineering the infrastructure so that partitions are so rare
and brief that the "unavailability window" is measured in seconds per year.

**Key numbers:**
- TrueTime uncertainty: ~1–7ms typical, bounded at ~14ms
- Spanner external read latency: ~10–100ms (must wait out TrueTime uncertainty for reads)
- Spanner global write latency: ~tens of ms within a region, ~100ms cross-continent

---

**Q: What are the real-world implications of CAP for database selection?**

Use this as a decision framework:

**"What happens if two nodes disagree for 5 seconds?"**

- If the answer is "that's a serious bug" (banking, inventory, locks) → choose CP.
- If the answer is "that's fine, we'll reconcile" (social, analytics, carts) → choose AP.

**Common mappings:**

| Use case | Choice | Reasoning |
|----------|--------|-----------|
| Bank account balance | CP (PostgreSQL, Spanner) | Never show wrong balance |
| Flight seat reservation | CP | Never double-sell a seat |
| User profile reads | AP (DynamoDB, Cassandra) | Stale name for 1s is fine |
| Social media likes/views | AP | Approximate counts are acceptable |
| Shopping cart | AP (DynamoDB) | Amazon accepts cart inconsistency over unavailability |
| Distributed lock / leader election | CP (etcd, ZooKeeper) | Two leaders is catastrophic |
| DNS record propagation | AP | Stale records for minutes are tolerated globally |
| Session tokens / auth cache | Depends | AP usually fine; for revocation, CP matters |

---

**Q: What is a common misconception about CAP that trips people up in interviews?**

**Misconception 1: "CA systems exist."**
A CA system would be consistent and available but not partition-tolerant. In a single-node
database (no distribution), there is no partition to tolerate — so a single-node PostgreSQL
instance is "CA." But the moment you add a replica or any second node, you have a distributed
system and partitions become possible. You must choose CP or AP. There are no CA distributed
databases.

**Misconception 2: "CAP means you always lose one property."**
CAP only constrains behaviour *during a partition*. When the network is healthy, a well-built
AP system can *also* provide strong consistency (e.g., Cassandra with QUORUM on both reads
and writes). CAP is not a permanent sacrifice — it defines behaviour under failure.

**Misconception 3: "Availability means 100% uptime."**
In CAP, availability means every non-failed node returns a response. A system where a
majority of nodes return responses but a minority partition is down is still considered
unavailable in CAP terms (for those nodes). This is stricter than the SRE definition of
availability (uptime percentage).

**Misconception 4: "Eventual consistency means data can be wrong forever."**
Eventual consistency guarantees convergence — given no new writes, replicas *will* converge.
In practice, Cassandra and DynamoDB converge in milliseconds to seconds. The window of
staleness is brief and bounded in well-operated systems.

---

**Q: How does CAP interact with the concept of a "split-brain" scenario?**

A **split-brain** occurs in a CP system when partition-handling logic fails and *both*
sides of a partition believe they are the primary/leader. This is the worst-case failure
for a CP system — you lose the "C" guarantee while also losing availability for some nodes.

**Example:** A two-node primary-replica cluster with a network partition. If the replica
doesn't hear from the primary, it may promote itself to primary. Now both nodes are accepting
writes as primary. When the partition heals, you have conflicting writes on both nodes with
no clean way to merge.

**How systems prevent split-brain:**
- **Quorum / odd-node clusters:** Always deploy 3, 5, or 7 nodes. A quorum of `(N/2)+1`
  is required to elect a leader. With 3 nodes, a single-node partition cannot form a quorum
  and is forced to step down. The majority side retains the leader.
- **Fencing tokens:** The old leader is given a token. Any write to storage must include
  this token. When a new leader is elected with a higher token, storage rejects writes from
  the old leader, even if it hasn't noticed it was deposed.
- **STONITH (Shoot The Other Node In The Head):** The new leader instructs infrastructure
  (cloud API, power controller) to forcibly kill or isolate the old node before assuming
  leadership.

**Rule of thumb:** Never run a replicated stateful system with an even number of nodes
(2, 4, 6). Even-numbered clusters have no quorum majority during a symmetric partition and
are maximally vulnerable to split-brain.