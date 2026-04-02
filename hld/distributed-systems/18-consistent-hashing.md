# 18. Consistent Hashing

## What is Consistent Hashing?

Consistent hashing is a technique for distributing keys across nodes such that when a node is added or removed, **only K/N keys need to be remapped** (where K = total keys, N = number of nodes). In naïve modular hashing (hash % N), almost all keys remap when N changes.

---

## Q1: What's the problem with naive hash-based sharding?

```
Nodes: A, B, C  (N=3)
key1 → hash(key1) % 3 = node A
key2 → hash(key2) % 3 = node C

Add node D (N=4):
key1 → hash(key1) % 4 = node B  ← MOVED
key2 → hash(key2) % 4 = node A  ← MOVED
```

Almost every key remaps. For a cache, this means a **cache miss storm** — all requests suddenly miss and hit the database. For a distributed database, it means massive data migration.

---

## Q2: How does consistent hashing work?

### The Ring

1. Map the hash space (0 to 2^32 - 1) onto a **ring** (circular).
2. Hash each **node** to a position on the ring.
3. Hash each **key** to a position on the ring.
4. A key is assigned to the **first node clockwise** from its position.

```
         0
      /     \
  Node C    Node A
    |           |
  Node B ——————
         2^32/2
```

**Example:**
```
Ring positions (mod 360 for simplicity):
Node A → 90°
Node B → 180°
Node C → 270°

Key X → 120° → first node clockwise → Node B
Key Y → 200° → first node clockwise → Node C
Key Z → 300° → first node clockwise → Node A (wraps around)
```

### Adding a Node

Add Node D at 150°:
- Keys between 120° and 150° (previously going to Node B) now go to Node D.
- Only those keys need to migrate. All others stay.

**Impact:** Only ~1/N of keys are affected.

### Removing a Node

Remove Node B at 180°:
- Keys that were on Node B now go to Node C (next clockwise).
- Only ~1/N of keys need to migrate.

---

## Q3: What are virtual nodes (vnodes) and why are they needed?

### The Problem with Basic Consistent Hashing

With only one point per node on the ring, distribution can be uneven:
- Node A might cover 10% of the ring
- Node B might cover 60% of the ring
- Node C might cover 30% of the ring

Also, when a node is removed, all its load goes to the single next node — unbalanced.

### Virtual Nodes (Vnodes)

Each physical node is assigned **multiple positions** on the ring (virtual nodes). For example, Node A might have 100 virtual positions: A1, A2, ..., A100 scattered around the ring.

```
Ring: ...A3...B1...C2...A1...B3...C1...A2...B2...C3...
```

**Benefits:**
1. **More even distribution** — with 100-256 vnodes per node, statistical distribution is much smoother
2. **Graceful load redistribution** — when a node is added/removed, it takes/gives a small portion from *every* other node (not just its neighbor)
3. **Handles heterogeneous nodes** — a more powerful node gets more vnodes

**Trade-off:** More metadata to track (which vnodes belong to which physical node).

**In practice:** Cassandra uses 256 vnodes per node by default.

---

## Q4: How does consistent hashing work with replication?

For replication factor R, a key is stored on R consecutive nodes clockwise from its hash position.

```
N=5 nodes, R=3:
Key X hashes to position between Node 2 and Node 3:
  Primary: Node 3
  Replica: Node 4
  Replica: Node 5
```

When reading with quorum (R=3, W=2, Q=2):
- Read from Node 3, 4, 5 → wait for 2 responses → take latest version

**Cassandra's implementation:**
- Replication strategy defines which nodes get copies
- NetworkTopologyStrategy ensures replicas are in different racks/datacenters

---

## Q5: Where is consistent hashing used in real systems?

| System | Usage |
|--------|-------|
| **Cassandra** | Partitioning data across nodes; 256 vnodes/node |
| **DynamoDB** | Distributed key-value storage, paper describes consistent hashing |
| **Memcached** | Client-side consistent hashing for cache key routing |
| **Redis Cluster** | Uses 16,384 hash slots (a form of fixed virtual sharding, similar concept) |
| **CDN (Akamai, etc.)** | Route requests to cache servers |
| **Load balancers** | Consistent hashing for session affinity without sticky sessions per IP |

---

## Q6: What is a hash slot (Redis Cluster variant)?

Redis Cluster uses a variation: 16,384 fixed **hash slots**.

```
key → CRC16(key) % 16384 → slot number
```

Each node owns a range of slots. To add a node, move some slots (with their data) to the new node. This is deterministic and controllable.

**vs. consistent hashing:**
- Hash slots: Fixed slot count, manually or automatically reassigned
- Consistent hashing: Ring-based, vnodes add probabilistic balance

Both achieve the same goal: minimize data movement on topology changes.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Fraction of keys remapped on node add/remove | ~1/N |
| Cassandra vnodes per node | 256 (default) |
| Redis hash slots | 16,384 |
| Consistent hashing ring size | 0 to 2^32 - 1 |
| Keys remapped (N=10) when 1 node added | ~10% |

---

## Interview Q&A

**Q: Why use consistent hashing for a distributed cache instead of hash % N?**
A: With hash % N, adding or removing a cache node invalidates ~(N-1)/N of cache entries — almost everything misses on the DB. Consistent hashing limits remapping to ~1/N of keys. For a cache serving millions of QPS, that difference is the difference between a smooth scale-up and a thundering herd that takes down the database.

**Q: How do virtual nodes solve the uneven distribution problem?**
A: A single point per node on the ring creates uneven arc lengths (some nodes own more of the keyspace). With 100-256 virtual nodes per physical node, the law of large numbers kicks in — each node ends up owning approximately equal portions of the keyspace. When nodes are added/removed, the load redistributes across all remaining nodes proportionally.

**Q: What's the downside of too many virtual nodes?**
A: Memory overhead to track all vnode-to-node mappings, and more gossip traffic to propagate membership changes. Also, during node addition, you're doing many small data transfers instead of one large one — more overhead per byte transferred. Cassandra found 256 to be a practical sweet spot.