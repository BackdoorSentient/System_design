# 19. Consensus Algorithms — Raft & Paxos

## What is Consensus?

In a distributed system, **consensus** means getting a group of nodes to agree on a single value (or sequence of values) even if some nodes fail or messages are delayed.

**Why is it hard?** The FLP impossibility theorem (1985) proves that in an asynchronous system with even one faulty node, consensus cannot be guaranteed to terminate. In practice, we work around this with timeouts and partial synchrony assumptions.

**Where consensus is used:**
- Electing a leader in a replicated database
- Committing a transaction across distributed nodes
- Agreeing on the next log entry in a replicated state machine (etcd, ZooKeeper, CockroachDB)

---

## Q1: What is the Raft consensus algorithm?

Raft was designed to be **understandable** — it decomposes consensus into relatively independent sub-problems.

### Key Roles

| Role | Description |
|------|-------------|
| **Leader** | Handles all client requests; replicates log to followers |
| **Follower** | Passive; responds to leader and candidate RPCs |
| **Candidate** | Transitional state during leader election |

### Raft Terms

The timeline is divided into **terms** — monotonically increasing integers. Each term begins with an election. Terms act as a logical clock.

```
Term 1: Leader A elected → operates
Term 2: A fails → election → Leader B elected → operates
Term 3: Network partition → split votes → no leader → Term 4...
```

### Leader Election

1. Follower receives no heartbeat for a random **election timeout** (150–300ms)
2. Follower increments term, votes for itself, sends `RequestVote` RPCs to all others
3. **Wins election** if majority votes received
4. Sends heartbeats (empty `AppendEntries` RPCs) to prevent new elections

**Why random timeout?** Prevents all followers from starting elections simultaneously, reducing split-vote scenarios.

### Log Replication

1. Client sends command to leader
2. Leader appends command to its log (uncommitted)
3. Leader sends `AppendEntries` RPCs to all followers
4. When majority acknowledges, leader **commits** the entry
5. Leader applies to state machine, responds to client
6. Next heartbeat tells followers to commit

```
Leader:   [1|set x=1] [2|set y=2] [3|del z] ← committed
Follower: [1|set x=1] [2|set y=2]            ← catching up
```

### Safety Guarantees

- **Election safety:** At most one leader per term
- **Log matching:** If two logs have same index + term, all preceding entries are identical
- **Leader completeness:** A leader has all committed entries from previous terms
- **State machine safety:** All servers apply same entry at same log index

### Raft Limitations

- Requires a majority (n/2 + 1) of nodes to be operational
- Single leader is a write bottleneck
- Leader must be in the majority partition during a network split (CP, not AP)

**Used by:** etcd, CockroachDB, TiKV, Consul, RethinkDB

---

## Q2: What is Paxos (conceptually)?

Paxos, invented by Leslie Lamport, is the theoretical foundation of consensus. It's notoriously difficult to understand and implement correctly.

### Roles

| Role | Description |
|------|-------------|
| **Proposer** | Proposes a value to be agreed upon |
| **Acceptor** | Votes on proposals |
| **Learner** | Learns the agreed value |

### Two Phases

**Phase 1: Prepare**
1. Proposer chooses proposal number N (must be higher than any seen)
2. Sends `Prepare(N)` to majority of acceptors
3. Acceptors promise not to accept proposals < N; respond with the highest proposal they've already accepted

**Phase 2: Accept**
1. If proposer receives majority of promises, it sends `Accept(N, value)`
   - Value is from the highest-numbered accepted proposal in responses, or proposer's own value if none
2. Acceptors accept the proposal unless they've promised to a higher N
3. If majority accepts → **consensus reached**

### Why Paxos is Hard

- The basic protocol only covers a single-value consensus (single-decree Paxos)
- **Multi-Paxos** (for a sequence of values, like a log) requires additional leader election and optimization that Lamport described only informally
- Liveness: if two proposers keep outbidding each other, no progress is made (leader needed)
- Real implementation (Google Chubby, Zookeeper's ZAB) required significant new engineering

### Paxos vs Raft

| | Paxos | Raft |
|---|---|---|
| Designed for | Theoretical correctness | Understandability |
| Leader election | Not prescribed | Integrated |
| Log ordering | Not prescribed | Core to design |
| Implementations | Chubby, ZooKeeper (ZAB), Spanner | etcd, CockroachDB, Consul |
| Learning curve | Very high | Moderate |

---

## Q3: What is a replicated state machine?

Consensus algorithms are used to implement **replicated state machines** — a cluster where every node applies the same sequence of commands and ends up in the same state.

```
Client → [Leader] → Log: [cmd1, cmd2, cmd3, ...]
                        ↓ (via consensus)
         [Follower 1] → Log: [cmd1, cmd2, cmd3, ...]  → same state
         [Follower 2] → Log: [cmd1, cmd2, cmd3, ...]  → same state
```

**Used for:** Distributed config stores (etcd), distributed locks (Chubby), database replication (CockroachDB).

---

## Q4: What guarantees does consensus provide?

| Guarantee | Description |
|-----------|-------------|
| **Safety** | Nothing bad happens — all nodes agree on the same value |
| **Liveness** | Something good eventually happens — consensus is eventually reached |
| **Fault tolerance** | Works as long as majority (n/2+1) of nodes are operational |

A Raft cluster of 5 nodes tolerates 2 failures. A cluster of 3 tolerates 1 failure.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Raft election timeout | 150–300ms (randomized) |
| Raft heartbeat interval | ~50ms (must be << election timeout) |
| Majority required (N nodes) | N/2 + 1 |
| Raft cluster size for 2 failure tolerance | 5 nodes |
| Raft cluster size for 1 failure tolerance | 3 nodes |
| etcd max recommended cluster size | 7 nodes |

---

## Real-World Examples

| System | Algorithm |
|--------|-----------|
| etcd | Raft |
| CockroachDB | Raft (per range) |
| TiKV (TiDB) | Raft |
| Consul | Raft |
| Google Chubby | Paxos |
| Apache ZooKeeper | ZAB (Zookeeper Atomic Broadcast, Paxos-inspired) |
| Google Spanner | Paxos |

---

## Interview Q&A

**Q: Why does Raft require a majority, not just any quorum?**
A: Any two majorities of N nodes must overlap by at least one node. This overlap node guarantees that any newly elected leader has seen all committed entries from the previous term — it prevents data loss on leader failover. If we used a non-overlapping quorum, a new leader might not know about committed entries.

**Q: What happens in Raft during a network partition?**
A: The partition with a majority of nodes continues to elect a leader and make progress (CP system). The minority partition can't elect a leader (can't get majority votes) and goes read-only (stale reads possible). When partition heals, the minority nodes sync up with the new leader's log, discarding any conflicting entries.

**Q: When would you use Raft vs just a single leader with async replication?**
A: Raft gives you strong consistency and automatic failover — use it for metadata, configuration, or coordination where correctness is paramount (etcd, Zookeeper use cases). For high-throughput data replication where some eventual consistency is acceptable (read replicas, caching), async replication has far better performance and is sufficient.