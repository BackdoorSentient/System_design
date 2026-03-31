# Consistency Models

**Q: Why does consistency matter in distributed systems?**
A: In a distributed system, data is replicated across multiple nodes for availability and
performance. When you write to one node, the other nodes need to sync that write. During that
sync window — which can be milliseconds to seconds — different nodes hold different values.
Consistency models define the rules: what can a client read, and when?

Different applications have different needs. A bank cannot tolerate stale balance reads.
A social media feed can tolerate showing a post that's a few seconds old.

**Q: What is strong (linearizable) consistency?**
A: Strong consistency (also called linearizability) guarantees that the system behaves as if
there is only one copy of the data. After a write completes, every subsequent read from any
node returns that value. Operations appear to take effect atomically at a single point in time.

How it's achieved: Before acknowledging a write as complete, the system waits for a quorum
(majority) of replicas to confirm they've written the value. Reads also query a quorum to
ensure they see the latest write.

Cost: Higher write latency (must wait for quorum), reduced availability (if quorum isn't
reachable, the system refuses reads/writes rather than risk serving stale data).

Used in: Traditional relational databases (PostgreSQL, MySQL with synchronous replication),
ZooKeeper, etcd, Google Spanner, financial transaction systems.

**Q: What is eventual consistency?**
A: Eventual consistency guarantees that if no new writes occur, all replicas will eventually
converge to the same value. Reads in the meantime may return stale data.

How it works: Writes are acknowledged as soon as one node (or a small subset) accepts them.
Other replicas sync asynchronously in the background. There is no guaranteed time bound on
convergence, though in practice it's often milliseconds to seconds.

The benefit is very low write latency and high availability — the system accepts writes even
when some replicas are unreachable.

Conflict resolution: When two nodes independently accept conflicting writes (during a network
partition), the system must resolve conflicts. Strategies include last-write-wins (based on
timestamp), vector clocks, or application-level merge logic (Amazon shopping cart uses this).

Used in: Cassandra, DynamoDB, CouchDB, DNS, Amazon S3 (though S3 now offers strong
consistency), CDN edge caches.

**Q: What is causal consistency?**
A: Causal consistency guarantees that operations that are causally related are seen in causal
order by all nodes. Operations with no causal relationship may be seen in any order.

Two events are causally related if: A happens before B, or A and B both depend on a common
prior event.

Example: User A posts a message (event 1). User A then posts a reply "As I said above..."
(event 2, causally depends on event 1). Causal consistency guarantees no one sees the reply
before the original message. But two unrelated posts from different users may appear in
different orders on different nodes.

Implementation: Vector clocks or logical timestamps track causality. A node delays delivering
a message until all its causal predecessors have been received.

Used in: Distributed collaborative editing (Google Docs uses a variant), social media
comment threads, messaging apps.

**Q: What is read-your-writes consistency?**
A: Read-your-writes (also called read-your-own-writes) guarantees that after you perform a
write, your own subsequent reads will always reflect that write — even if other users may
not see it yet.

This is crucial for UX. If you post a comment and then immediately reload the page, you
expect to see your comment. Without this guarantee, you'd post and then see the page without
your comment, making you think it failed.

Implementation: Route all reads for a user to the same replica they wrote to, or include a
version token in the write response that read requests use to wait for the correct replica
version.

**Q: What is monotonic read consistency?**
A: Monotonic read consistency guarantees that if a client reads a value, subsequent reads will
never return an older value. You won't see time go backward.

Without it: You read a post with 100 likes, then refresh and see 95 likes (because your
second read hit a less-synced replica). This is disorienting.

With it: Each read is at least as fresh as the previous one. Achieved by routing a client's
reads to the same replica, or by tracking read versions.

**Q: What are the consistency levels in Cassandra and how do you choose?**
A: Cassandra lets you tune consistency per-query. Common levels:

ONE: Read/write succeeds if 1 replica responds. Fastest, least consistent. Good for
non-critical metrics or logs.

QUORUM: Read/write succeeds when majority of replicas respond (e.g. 2 of 3). Balances
consistency and availability. The sweet spot for most applications.

ALL: Every replica must respond. Strongest consistency, but any replica failure causes
failure. Only use for critical writes where you truly cannot afford any inconsistency.

LOCAL_QUORUM: Quorum within the local data center only. Good for multi-region deployments
where cross-region latency would be unacceptable.

The key insight: if you use QUORUM for both reads and writes, and your replication factor
is N, then read quorum + write quorum > N — meaning at least one node always has the latest
write. This gives you strong consistency without needing ALL.

**Q: What is the difference between consistency in CAP theorem vs ACID transactions?**
A: These use the word "consistency" differently — a source of much confusion:

CAP consistency (linearizability): About distributed replication — all nodes see the same
data at the same time. A property of the distributed system's replication protocol.

ACID consistency: About database integrity constraints — a transaction brings the database
from one valid state to another valid state. Foreign keys are not violated, constraints hold.
This is enforced by the database engine, not the replication protocol.

A database can have ACID consistency (valid constraints) while being eventually consistent
in the CAP sense (replicas may temporarily diverge). These are orthogonal properties.