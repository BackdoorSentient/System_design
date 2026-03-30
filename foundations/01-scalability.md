# Scalability

**Q: What is scalability?**
A: Scalability is a system's ability to handle growing load — more users, more data, more requests —
without degrading performance or requiring a complete redesign. A scalable system can grow
gracefully. There are two dimensions: scaling to handle more traffic (load scalability) and scaling
to handle more data (data scalability).

**Q: What is vertical scaling and what are its limits?**
A: Vertical scaling means upgrading the hardware of a single machine — more CPU cores, more RAM,
faster SSDs, better network cards. It is simple because your application code doesn't change —
you just run the same software on a more powerful machine.

Limits:
- Hardware ceiling: There is a maximum RAM and CPU you can put in one machine. As of today,
  even the largest cloud instances top out around 24TB RAM and 448 vCPUs.
- Cost curve: Doubling compute on one machine costs far more than double — high-end hardware is
  exponentially expensive.
- Downtime: Upgrading a machine usually requires taking it offline.
- Single point of failure: One machine means one failure brings everything down.
- No redundancy: If the machine's disk fails, data can be lost.

Good use cases: Small-to-medium databases, legacy monoliths that can't be distributed,
administrative tools with low traffic.

**Q: What is horizontal scaling and how does it work?**
A: Horizontal scaling means adding more machines (nodes) and distributing work across them.
Instead of one powerful server, you have many commodity servers working in parallel.

How it works:
- A load balancer sits in front of all servers and routes each incoming request to one of them.
- Each server runs the same application code.
- Servers are stateless — they don't store session data locally. State lives in a shared database
  or cache (like Redis) so any server can handle any request.
- You can add or remove servers dynamically based on traffic (auto-scaling).

Benefits: No hardware ceiling, fault tolerant (losing one server doesn't kill the system),
cheaper at scale (commodity hardware), enables geographic distribution.

Challenges: Requires stateless application design, introduces distributed systems complexity,
needs a load balancer, data consistency becomes harder.

**Q: What does it mean for a service to be stateless, and why does it matter for scaling?**
A: A stateless service does not store any request-specific data between requests on the server
itself. Every request carries all the information needed to process it (e.g. a JWT token for
auth), or that information lives in a shared external store (database, Redis cache).

Why it matters: If Server A handles your login and stores your session in memory, then Server B
gets your next request and doesn't know who you are — you appear logged out. Stateless design
solves this by storing session data in a shared Redis cluster. Now any server can handle any
request. This is what makes horizontal scaling possible.

**Q: What is a load balancer and what algorithms does it use?**
A: A load balancer is a component that distributes incoming traffic across multiple backend
servers. It also performs health checks and stops routing to failed servers.

Common algorithms:
- Round robin: Requests go to servers in rotation (1→2→3→1→2→3). Simple, works when all
  servers are identical.
- Weighted round robin: Servers with more capacity get more requests. Useful when servers
  have different specs.
- Least connections: New request goes to the server with fewest active connections. Better for
  requests with variable processing time.
- IP hash: Client's IP is hashed to always route to the same server. Useful for sticky sessions
  but breaks if servers are added/removed.
- Random: Requests are assigned randomly. Statistically balances well at high volume.

Load balancers can operate at Layer 4 (TCP — routing by IP/port) or Layer 7 (HTTP — routing by
URL path, headers, cookies). Layer 7 is more flexible but more CPU intensive.

**Q: What is auto-scaling?**
A: Auto-scaling automatically adds or removes servers based on real-time metrics like CPU
utilization, QPS, or memory usage. Cloud providers (AWS, GCP, Azure) offer this natively.

Scale-out trigger example: If average CPU across all servers exceeds 70% for 2 minutes,
add 2 more servers.
Scale-in trigger example: If average CPU drops below 30% for 10 minutes, remove 2 servers.

This lets you handle traffic spikes (flash sales, viral moments) without over-provisioning
permanently, which saves money.

**Q: What is the difference between scaling reads and scaling writes?**
A: Read scaling is easier. You add read replicas of your database — copies that stay in sync
with the primary and serve read queries. You can have many replicas.

Write scaling is harder. All writes must go to the primary (or be coordinated across nodes),
which is a bottleneck. Solutions include: sharding (splitting data across multiple databases),
write-optimized databases (Cassandra), command-query responsibility segregation (CQRS), and
event sourcing.

Most real systems are read-heavy (Twitter: reads >> writes), so read replicas + caching solves
most scaling problems.

**Q: What is database sharding?**
A: Sharding means splitting your database horizontally across multiple machines, where each
machine (shard) holds a subset of the data. For example, users with IDs 1–1M go to Shard A,
1M–2M go to Shard B, etc.

Benefits: Each shard handles a fraction of the total load. You can scale writes by adding shards.

Problems:
- Cross-shard queries are expensive (joining data across machines).
- Resharding is painful — adding a new shard means redistributing data.
- Hot shards: If shard A gets all the famous users, it becomes a bottleneck.
- No cross-shard transactions with ACID guarantees.

Sharding is a last resort — try caching, read replicas, and query optimization first.

**Q: What is the N+1 problem and why does it kill scalability?**
A: The N+1 problem occurs when code makes 1 query to get a list of N items, then makes N
additional queries to get details for each item — totalling N+1 database queries.

Example: Fetch 100 posts (1 query), then for each post fetch its author (100 queries) = 101
queries instead of 1 JOIN. At scale this destroys database performance. Fix it with eager
loading, JOIN queries, or DataLoader-style batching.