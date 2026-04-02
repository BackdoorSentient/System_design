# 28. Gossip Protocol

## What is the Gossip Protocol?

The gossip protocol (also called epidemic protocol) is a peer-to-peer communication method where each node **periodically shares information with a few random neighbors**. Information spreads through the cluster like a rumor — each node that learns something new spreads it to others.

**Analogy:** Imagine you start a rumor in a room. You tell 2 random people. Each of them tells 2 more. By round 3–4, nearly everyone in the room knows.

**Why gossip?**
- O(log N) dissemination time for N nodes
- No single point of failure (no central coordinator)
- Self-healing — if one path fails, information finds another path
- Scales to thousands of nodes

---

## Q1: How does the gossip protocol work?

### Basic Algorithm

Every T seconds (gossip interval, typically 1s), each node:
1. Picks K random nodes from its membership list (typically K=2 or K=3)
2. Sends its current state (or state deltas) to those K nodes
3. Receiving nodes merge the information with their own state
4. Process repeats

```
Round 1:  A tells B, C          (A knows)
Round 2:  B tells D, E          (A, B know)
           C tells F, G          (A, C know)
Round 3:  D tells H, A          (A, B, D know)
           ...etc
```

After log₂(N) rounds, all N nodes have the information.

### Three Gossip Variants

**Push:** Nodes push their state to K random neighbors.
- Efficient for state that changes rarely
- Propagation: O(log N) rounds

**Pull:** Nodes ask K random neighbors for their state.
- Good for catching up nodes that were partitioned

**Push-Pull:** Exchange state both ways in each gossip.
- Best convergence time — used by Cassandra, Dynamo

---

## Q2: What does gossip carry?

Gossip can propagate different types of information:

### 1. Membership (Who is alive?)
Each node maintains a **membership list** with heartbeat counters for every other node.

```
Node A's view:
  Node A: heartbeat=1042, timestamp=T-0s
  Node B: heartbeat=998,  timestamp=T-2s
  Node C: heartbeat=100,  timestamp=T-300s  ← suspect (stale)
```

If a node's heartbeat hasn't been updated in a threshold time → marked **suspect** → eventually **dead**.

### 2. Application State
Cassandra uses gossip to propagate not just heartbeats but metadata:
- Current token ranges (consistent hash ring position)
- Schema version
- Data center/rack topology
- Snitch information

### 3. Configuration Updates
New cluster configuration (added/removed nodes, rack changes) propagates via gossip.

---

## Q3: How does Cassandra use gossip?

Cassandra runs gossip every **1 second**, exchanging with **1–3 random peers** using push-pull gossip.

**GossipDigestSyn → GossipDigestAck → GossipDigestAck2**

```
Node A → B: GossipDigestSyn(A: gen=1 ver=100, C: gen=1 ver=50)
Node B → A: GossipDigestAck(
              B: gen=1 ver=200,    ← B's own state (A asked for it)
              diff for A/C         ← what B knows that's newer
            )
Node A → B: GossipDigestAck2(
              updated state for nodes B didn't have
            )
```

**What Cassandra gossip propagates:**
- `STATUS` — NORMAL, LEAVING, LEFT, REMOVING
- `LOAD` — disk usage for load balancing
- `SCHEMA` — schema version digest
- `RACK`, `DC` — topology info

---

## Q4: How quickly does gossip converge?

For N nodes with K=2 peers per gossip round, convergence time is:

```
rounds ≈ log₂(N) + log₂(log₂(N))
```

| N (nodes) | Rounds to converge |
|-----------|-------------------|
| 10        | ~4 rounds         |
| 100       | ~7 rounds         |
| 1000      | ~10 rounds        |
| 10000     | ~14 rounds        |

With a 1-second gossip interval, a 1000-node cluster converges in ~10 seconds.

---

## Q5: What are the trade-offs of gossip?

### Pros

1. **Scalability** — O(log N) convergence, O(N) total messages per round (each node sends to K neighbors)
2. **Fault tolerant** — no SPOF; even if many nodes fail, gossip routes around them
3. **Self-healing** — new nodes automatically learn cluster state
4. **Decentralized** — no central coordinator needed

### Cons

1. **Eventual consistency** — information propagates slowly compared to broadcast; stale state during propagation window
2. **Bandwidth overhead** — N nodes × K messages × state_size per round. For large state, compressing with digests is essential.
3. **False positives** — a slow node looks like a dead node during gossip delay
4. **No total ordering** — gossip doesn't guarantee everyone sees updates in the same order

---

## Q6: Gossip vs Broadcast vs Consensus

| | Broadcast | Gossip | Consensus (Raft) |
|---|---|---|---|
| Consistency | Strong (if acked) | Eventual | Strong |
| SPOF | Coordinator | None | Leader (but replicated) |
| Messages per update | O(N) | O(N log N) total | O(N) per round × rounds |
| Scale | Poor (coordinator bottleneck) | Excellent | Good (up to ~7–9 nodes) |
| Use case | Small clusters, strong consistency | Large clusters, membership | Small critical clusters |

---

## Real-World Examples

| System | Gossip Usage |
|--------|-------------|
| **Cassandra** | Membership, token ranges, schema version |
| **DynamoDB** | Node membership (described in Dynamo paper) |
| **Riak** | Ring membership |
| **Consul** | Node failure detection (SWIM protocol) |
| **Bitcoin P2P** | Transaction and block propagation |
| **HashiCorp Serf** | Gossip library (used by Consul) |

---

## Q7: What is the SWIM Protocol?

**SWIM (Scalable Weakly-consistent Infection-style Membership)** is a gossip-based failure detector that reduces false positives.

**Instead of timeout-based detection:**
1. Node A sends `PING` to random node B
2. If no `ACK` from B within timeout, A asks K random nodes to `PING-REQ B` (indirect probe)
3. If none get an `ACK` either → B is marked **suspect**
4. If B doesn't refute within a grace period → B is marked **faulty** → gossiped to all

**Why indirect probe?** If A→B is failing but C→B works, B might be alive but A's connection to B is broken. The indirect probe distinguishes "B is down" from "A→B link is down."

**Used by:** HashiCorp Consul (via Serf), Kubernetes (custom implementations)

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Cassandra gossip interval | 1 second |
| Cassandra gossip peers per round | 1–3 |
| Convergence time (1000 nodes) | ~10 seconds |
| SWIM probe timeout | 500ms typical |
| Cassandra phi failure threshold | φ = 8 |

---

## Interview Q&A

**Q: Why doesn't Cassandra use Raft for cluster membership instead of gossip?**
A: Raft provides strong consistency but doesn't scale beyond ~7–9 nodes practically (O(N) messages per consensus round becomes expensive at 100+ nodes). Cassandra clusters can have hundreds of nodes. Gossip scales to this with O(log N) convergence and doesn't require a majority quorum to make progress. The trade-off is eventual consistency in membership — acceptable for a database whose consistency model is already tunable.

**Q: How does gossip handle network partitions?**
A: Gossip is resilient to partitions. Nodes in each partition continue gossiping among themselves. Nodes in the other partition will eventually be marked suspect/dead by both sides. When the partition heals, gossip quickly converges the merged views (push-pull exchange fills in what was missed). No operator intervention needed.

**Q: What's the difference between gossip for membership vs gossip for data?**
A: Membership gossip carries small, bounded state (node status, heartbeat counters) — easy to gossip the full membership list. Data gossip (like Bitcoin transactions) carries unbounded data; you can't gossip all of it to all peers. Bitcoin uses an inventory/request model: gossip that "I have TX #abc," recipients who don't have it request it. The gossip layer handles discovery; the actual data transfer is point-to-point.