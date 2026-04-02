# 26. Heartbeats, Timeouts & Failure Detection

## Why is Failure Detection Hard?

In distributed systems, you cannot distinguish between a **crashed node** and a **slow node** or a **network partition**. A node that doesn't respond might be:
- Dead (process crashed, hardware failure)
- Slow (overloaded, GC pause)
- Network isolated (can still serve local requests)

**The fundamental challenge:** In an asynchronous network, there is no way to know with certainty that a node has failed — only that you haven't heard from it in a while.

---

## Q1: What is a heartbeat?

A **heartbeat** is a periodic message sent between nodes to indicate "I'm still alive."

### Push-based heartbeat
Node A sends a heartbeat to Node B every T seconds. If B doesn't hear from A for 3T, B considers A failed.

```
A → B: heartbeat (t=0)
A → B: heartbeat (t=5)
A → B: heartbeat (t=10)
[A crashes]
B waits until t=25 (3×heartbeat interval = 15s + timeout)
B declares A failed
```

### Pull-based heartbeat (health check)
B periodically polls A: "Are you alive?" If no response within timeout, A is considered failed.

```
B → A: GET /health (t=0) → 200 OK
B → A: GET /health (t=5) → 200 OK
B → A: GET /health (t=10) → timeout
B → A: GET /health (t=15) → timeout
B declares A failed after 2 consecutive failures
```

Used by: Load balancers (health checks), Kubernetes liveness probes, AWS ELB.

---

## Q2: How do you choose timeout values?

**Too short:** False positives — declare a slow-but-alive node as dead. Triggers unnecessary failover, creates instability.

**Too long:** Slow to detect real failures — users experience long outages before recovery kicks in.

**Common heuristic:**
```
timeout = mean_response_time + k × std_deviation_response_time
```

Where k = 4–8 gives low false positive rate for normally distributed latency.

**Typical values:**

| Layer | Timeout |
|-------|---------|
| Load balancer health check | 5–10s, fail after 3 consecutive misses |
| Kubernetes liveness probe | 10s interval, 3 failures → restart |
| Raft election timeout | 150–300ms |
| ZooKeeper session timeout | 30s |
| TCP keep-alive | 75s default, 30s recommended |
| Cassandra phi accrual threshold | φ = 8 (tunable) |

---

## Q3: What is the Phi (φ) Accrual Failure Detector?

Used by Cassandra and Akka. Instead of a binary "alive/dead" answer, the phi accrual detector outputs a **suspicion level φ** that increases over time without a heartbeat.

### How it works:

1. Track inter-arrival times of recent heartbeats (e.g., last 1000 heartbeats)
2. Fit a distribution (exponential or normal) to these times
3. When a heartbeat is late, compute:

```
φ(t) = -log10(P_later(t - t_last))
```

Where `P_later` = probability that the next heartbeat would arrive this late given the historical distribution.

**Interpretation:**
- φ = 1 → 10% probability of failure
- φ = 4 → 99.99% probability of failure  
- φ = 8 → 99.999999% probability of failure (Cassandra default threshold)

**Advantages over fixed timeouts:**
- Adapts to network jitter automatically (learns the expected distribution)
- Application can choose its own threshold (trade latency vs. false positive)
- Gradual suspicion rather than binary flip

**Used by:** Cassandra (membership), Akka cluster

---

## Q4: What is the split-brain problem?

**Split-brain:** A network partition causes two halves of a cluster to each think the other half is dead and both continue operating independently.

```
Cluster: [Node 1, Node 2, Node 3 | Node 4, Node 5]
          ← Partition A →        ← Partition B →

Both partitions declare themselves the leader/primary.
Both accept writes → data diverges.
Partition heals → data conflict.
```

**Consequences:**
- Two leaders writing to different DB replicas → data inconsistency
- Both cache clusters accepting writes → cache divergence

**Solutions:**

1. **Quorum:** Only the partition with majority (N/2 + 1) nodes continues operations. Minority partition goes read-only. (Raft, Paxos approach)

2. **STONITH (Shoot The Other Node In The Head):** When partition detected, the surviving partition fences (physically powers off or network-isolates) the other partition before taking over. Prevents both from operating simultaneously.

3. **Epoch/term numbers:** New leaders have higher epoch numbers. Old leader in minority partition gets rejected by storage (sees higher epoch in responses).

---

## Q5: What is the Gossip Protocol for failure detection?

(See also topic 28 for full Gossip coverage)

In large clusters (100s of nodes), each node can't heartbeat every other node — O(N²) messages. Gossip-based failure detection scales to large clusters.

**How it works:**
1. Each node maintains a **membership list** with the last-seen timestamp for every other node
2. Every T seconds, each node picks K random neighbors and **gossips** its membership list
3. Recipients merge the gossip into their local list (take newer timestamps)
4. If a node's timestamp isn't updated for TTL, it's marked suspect → eventually dead

Used by: Cassandra, DynamoDB, Riak for cluster membership.

---

## Q6: Kubernetes Liveness vs Readiness Probes

| | Liveness Probe | Readiness Probe |
|---|---|---|
| Question | "Is this pod alive?" | "Is this pod ready for traffic?" |
| On failure | Restart the pod | Remove pod from service endpoints |
| Use case | Detect deadlocks, infinite loops | Warm-up, temporary overload |
| Restart on failure? | Yes | No — just stops routing traffic |

**Example:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Cassandra φ accrual threshold | 8 (99.999999%) |
| Kubernetes liveness restart time | ~30–60s (3 failures × 10s interval) |
| Typical heartbeat interval | 1–10s |
| Raft election timeout | 150–300ms |
| TCP keep-alive (recommended) | 30s |
| ZooKeeper session timeout | 30s |

---

## Interview Q&A

**Q: Why is a fixed timeout not ideal for failure detection?**
A: Fixed timeouts don't account for variable network conditions. During high load or network congestion, response times increase — a fixed timeout set for normal conditions generates false positives (declares healthy nodes dead). The phi accrual detector learns the baseline response distribution and adjusts suspicion levels accordingly, giving far fewer false positives in fluctuating networks.

**Q: How does Kubernetes know when to restart a pod vs just remove it from the load balancer?**
A: Liveness probe failure → pod is restarted (it's considered unhealthy/deadlocked). Readiness probe failure → pod is removed from Service endpoints but NOT restarted (it's considered temporarily unready — e.g., warming up cache). This distinction prevents traffic from going to unready pods while allowing liveness issues to trigger recovery.

**Q: Two nodes both think the other is dead (split-brain). How do you resolve this?**
A: The canonical solution is quorum: only the partition with > N/2 nodes continues as primary. For 2-node clusters (no quorum possible), use an external tiebreaker: a "witness" third node or STONITH. Design systems to avoid even numbers of nodes — 3, 5, 7 enable quorum-based resolution.