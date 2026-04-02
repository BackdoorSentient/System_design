# 27. Leader Election

## What is Leader Election?

Leader election is the process by which distributed nodes agree on a single **coordinator (leader)** that takes responsibility for specific tasks.

**Why have a leader?**
- Simplifies coordination — only one node makes certain decisions
- Enables total ordering of operations
- Avoids conflicting updates (only leader writes)

**Examples:**
- One Kafka broker is the controller (manages partition assignments)
- One ZooKeeper node is the leader (processes all writes)
- One database node is the primary (all writes go here)
- One cron service instance runs a scheduled job at a time

---

## Q1: What algorithms are used for leader election?

### 1. Bully Algorithm

Nodes have IDs (higher ID = higher priority). When a node detects the leader is down:

1. It sends an `ELECTION` message to all nodes with higher IDs
2. If no response (higher IDs are all down), it declares itself leader and sends `COORDINATOR` to all lower IDs
3. If a higher-ID node responds, it takes over the election

```
Nodes: 1, 2, 3, 4, 5  (5 is current leader)
5 crashes. Node 2 starts election:
  2 → ELECTION → [3, 4] (not 5, it's assumed dead)
  3 responds: 3 → ELECTION → [4]
  4 responds: 4 → ELECTION → [] (no higher node alive)
  4 → COORDINATOR → [1, 2, 3]
  4 is new leader.
```

**Pros:** Simple, deterministic
**Cons:**
- O(N²) messages in worst case
- "Bully" — highest ID always wins, regardless of actual capability
- High ID node with poor connectivity might win but be unreliable

---

### 2. Ring-Based Election

Nodes are arranged logically in a ring. Each node knows its successor.

1. Detecting node sends `ELECTION(my_id)` to successor
2. Each node forwards to its successor, adding its own ID if it's larger
3. When the message travels the full ring and returns to the initiator with the max ID, that node declares itself leader
4. Sends `LEADER(max_id)` around the ring

```
Ring: 1 → 2 → 3 → 4 → 5 → 1
3 crashes. 2 detects, sends ELECTION(2):
  2 → ELECTION(2) → 4 (skipping dead 3)
  4 → ELECTION(4) → 5
  5 → ELECTION(5) → 1
  1 → ELECTION(5) → 2
  2 sees ELECTION(5) and 5 > 2 → 2 sends LEADER(5)
5 is new leader.
```

**Pros:** O(N) messages, simple
**Cons:** Slow (message travels full ring), O(N) time

---

### 3. Raft-Based Leader Election (Used in Practice)

(See Topic 19 for full Raft details)

**Key mechanism:**
- Nodes have randomized **election timeouts** (150–300ms)
- First node to time out sends `RequestVote(term)` to all others
- If it gets majority votes, it becomes leader for that term
- Leader sends heartbeats to prevent others from timing out

**Why it works well:**
- Randomized timeouts reduce simultaneous elections (split votes)
- Majority requirement prevents two leaders in same term
- Term numbers allow old leaders to recognize they've been superseded

**Used by:** etcd, CockroachDB, Consul, TiKV

---

### 4. ZooKeeper-Based Leader Election

**Mechanism:** Ephemeral sequential znodes.

1. All nodes create ephemeral sequential znodes: `/election/node-0000000001`, `-0000000002`, etc.
2. Node with **lowest sequence number** is the leader
3. All other nodes watch the znode just below theirs
4. When leader dies, its ephemeral znode disappears → next node gets notified → becomes leader

```
/election/
  node-0000000001  ← Leader (Process A)
  node-0000000002  ← Follower (Process B, watching 001)
  node-0000000003  ← Follower (Process C, watching 002)

A dies → 001 deleted → B gets watch notification → B is now leader
C is still watching 002 (still exists) → no unnecessary notification
```

**The "thundering herd" problem:** If all nodes watch the same znode (the leader), when leader dies, all nodes are notified simultaneously, causing many concurrent election attempts. The "watch only your predecessor" pattern avoids this.

**Used by:** Kafka (via ZooKeeper, pre-KRaft), many coordination use cases.

---

## Q2: What is split-vote / split-brain in leader election?

**Split vote:** During an election, no candidate gets majority → election restarts.
- In Raft: Multiple nodes time out simultaneously and split votes evenly. Raft uses randomized timeouts to resolve this statistically.

**Split-brain:** Two leaders exist simultaneously (bad!).
- Prevented by: requiring majority (quorum) — impossible for two groups to each have majority if total < N
- Raft guarantee: At most one leader per term due to majority rule

---

## Q3: Leader election in Kafka

**Pre-KRaft (ZooKeeper-based):**
- One broker is the **Controller** — manages partition leaders
- All brokers compete to create `/controller` ephemeral znode in ZooKeeper
- First to create wins → becomes Controller
- On Controller crash: ZooKeeper notifies all brokers → race to create new `/controller` znode

**KRaft (Kafka 3.x, ZooKeeper removed):**
- Kafka uses its own Raft implementation for controller election
- 3–5 controller nodes run Raft election among themselves
- Removes ZooKeeper dependency (simpler operations)

---

## Q4: Leader election trade-offs

| Algorithm | Messages | Time | Pros | Cons |
|-----------|----------|------|------|------|
| Bully | O(N²) | O(N) | Simple | High message cost |
| Ring | O(N) | O(N) | Efficient messages | Slow (ring traversal) |
| Raft | O(N) | O(1) terms | Fast, safe, practical | More complex |
| ZooKeeper | O(1) per node | Fast | Battle-tested | ZK dependency |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Raft election timeout | 150–300ms |
| ZooKeeper session timeout | 30s |
| Time to detect leader failure and elect new leader (Raft) | ~300–600ms |
| Kafka controller election time | ~30s (ZK-based), <1s (KRaft) |

---

## Interview Q&A

**Q: Why do you need leader election at all? Can't you just have all nodes coordinate via consensus every time?**
A: You can, but it's expensive. Every operation would require a multi-round consensus protocol. A leader amortizes this cost — the leader is elected once via consensus, and then makes decisions unilaterally (with followers just replicating). This trades election complexity for much faster normal-case operation. Leaderless protocols like Dynamo use quorum instead but accept weaker consistency.

**Q: What happens if the leader becomes a slow node (not crashed, just slow)?**
A: In Raft, a slow leader still sends heartbeats — followers won't time out and start elections. Operations queue up on the leader and are slow to complete. Clients may timeout waiting for responses. The leader won't be replaced unless it actually stops heartbeating. This is a reason to monitor leader response latency and sometimes manually trigger re-election. Some systems have a "step down" mechanism where a leader voluntarily resigns if it detects it's falling behind.

**Q: How is leader election different from distributed locking?**
A: Conceptually similar — both ensure only one node does something. Leader election is typically long-lived (leader serves for minutes/hours), built into the cluster's replication protocol, and tied to the cluster membership. Distributed locks are typically short-lived (seconds/minutes), used for application-level coordination (e.g., cron deduplication), and independent of cluster topology. ZooKeeper implements both using the same ephemeral sequential znode mechanism.