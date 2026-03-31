# 01_load_balancers.md — Load Balancers

---

**Q: What is a load balancer and why is it critical in distributed systems?**

A load balancer is a component that distributes incoming network traffic across a pool of backend servers to ensure no single server becomes a bottleneck. It acts as the single entry point for clients while hiding the horizontal scale of the backend.

Key purposes:
- **High availability**: If one server dies, traffic is routed to healthy ones.
- **Horizontal scalability**: Add servers without changing client configuration.
- **SSL termination**: Offload TLS decryption from app servers.
- **Observability**: Centralized point to collect metrics, logs, and traces.

Real-world: AWS ALB (Application Load Balancer), GCP Cloud Load Balancing, Nginx, HAProxy, F5 BIG-IP.

---

**Q: What is the difference between L4 and L7 load balancers? When do you use each?**

| Dimension | L4 (Transport Layer) | L7 (Application Layer) |
|---|---|---|
| Operates on | TCP/UDP packets | HTTP, gRPC, WebSocket payloads |
| Content awareness | IP + port only | Headers, URL, cookies, body |
| Speed | Faster — no payload parsing | Slower — full HTTP parsing |
| Routing granularity | By IP/port | By path, hostname, header, JWT claim |
| TLS handling | Pass-through (often) | Terminates TLS |
| Use cases | Raw TCP services, gaming, DB proxies | Web apps, microservices, API routing |

**L4 example**: You run a MySQL cluster and need a load balancer in front of read replicas. You use HAProxy at L4 — it sees TCP on port 3306 and round-robins connections without knowing SQL.

**L7 example**: You have `/api/users` routed to a Users service and `/api/products` routed to a Products service. An L7 LB (Nginx, ALB) inspects the URL path and routes accordingly.

**Trade-off**: L4 is ~2–5x lower latency per connection since it doesn't parse HTTP. L7 adds ~0.5–2ms overhead but enables content-based routing, A/B testing, canary deployments, and WAF integration.

---

**Q: Explain round robin, weighted round robin, and least connections algorithms. When does each break down?**

**Round Robin**
Requests are distributed sequentially: server 1 → server 2 → server 3 → server 1 → ...

- Assumes all requests are equal weight and all servers are identical.
- Breaks down when: requests have wildly different costs (a 100ms request vs a 10s request). A slow server builds up a queue while still receiving new requests at the same rate.

**Weighted Round Robin**
Each server has a weight. A server with weight 3 receives 3x the requests of a server with weight 1.

- Use when servers have different capacities (e.g., 32-core vs 8-core instances during a rolling upgrade).
- Still doesn't account for current load — a powerful server that happens to be doing expensive work can still get overloaded.

**Least Connections**
New requests go to the server with the fewest active connections.

- Works well for long-lived connections (WebSockets, streaming, databases).
- Overhead: the LB must track active connection counts per backend.
- Variant: **Weighted Least Connections** — balances by `active_connections / weight`.

**Least Response Time** (extension)
Routes to the server with the lowest combination of active connections and average response time. Used in Nginx Plus and HAProxy.

**Random**
Picks a server at random. Statistically converges to even distribution at scale. Avoids the herd problem where all LB nodes simultaneously agree on the "least loaded" server.

---

**Q: What are sticky sessions (session affinity)? What are the trade-offs?**

Sticky sessions ensure that requests from the same client always go to the same backend server. This is needed when session state is stored in server memory (not in a shared store like Redis).

**How it works**:
- **Cookie-based**: The LB injects a cookie (e.g., `AWSALB`) on the first response. Subsequent requests from that client include the cookie, and the LB uses it to route to the same backend.
- **IP-hash based**: Hash the client IP to consistently pick the same server. Breaks down behind NAT (millions of users sharing one IP all hit the same backend).

**Problems with sticky sessions**:
1. **Uneven load distribution**: If a "heavy" user (high traffic) is stuck to server 2, that server gets overloaded while others are idle.
2. **Node failures**: If the pinned server dies, the session is lost anyway (unless replicated).
3. **Scaling**: Adding a new server doesn't redistribute sticky sessions.
4. **Antipattern for stateless architectures**: The real fix is to externalize session state to Redis/DynamoDB so any server can handle any request.

**When to accept sticky sessions**: Legacy apps that cannot be refactored, or short-lived stickiness for cache warming (e.g., pin a user to a server for 60 seconds so their hot data is in L1 cache).

---

**Q: How does health checking work in load balancers, and what are active vs passive checks?**

**Active health checks**: The LB proactively sends test requests to backends on a schedule (e.g., every 5 seconds). A backend is marked unhealthy after N consecutive failures and removed from rotation. Restored after M consecutive successes.

Types:
- TCP check: Open a TCP connection to port 8080. If it succeeds, the server is alive.
- HTTP check: Send `GET /health HTTP/1.1` and expect a `200 OK`.
- Custom: Check response body for `{"status":"ok"}`.

**Passive health checks** (circuit breaker style): The LB observes real traffic. If a backend returns 5xx errors or times out on a configurable fraction of requests, it's temporarily ejected. Nginx calls this "slow start" + passive monitoring; Envoy calls it "outlier detection."

**Numbers that matter**:
- Typical health check interval: 5–30 seconds.
- Failure threshold: 2–3 consecutive failures before ejection.
- Recovery threshold: 2–3 successes before re-admission.
- Time to detect failure: interval × threshold = 10–90 seconds worst case.

For faster failure detection, use smaller intervals + TCP checks (cheaper than HTTP).

---

**Q: What is DNS-based load balancing and how does it compare to proxy-based LB?**

**DNS load balancing**: Multiple A records for the same domain. The DNS resolver returns different IPs in round-robin or geographically weighted fashion.

- Used by: AWS Route 53 (with latency, geolocation, weighted routing policies), Cloudflare.
- Pros: No single proxy bottleneck; globally distributed.
- Cons: TTL caching means changes take minutes to propagate; clients cache DNS aggressively, so "round robin" doesn't work at the request level — a client reuses the same IP for the TTL duration.

**Proxy-based (VIP) load balancing**: A virtual IP (VIP) is the single entry point. The LB process (Nginx, HAProxy, ALB) receives and forwards every packet.

- Pros: Per-request routing decisions, health checking, SSL termination, observability.
- Cons: Can itself become a bottleneck; must be made HA (active-passive or ECMP).

**In practice**: Both are used together. DNS routes users to the nearest datacenter (DNS LB). Within the datacenter, a proxy-based LB distributes across backend pods.

---

**Q: What is the Global Server Load Balancing (GSLB) pattern and how does it work?**

GSLB routes users to the best datacenter globally based on latency, geographic proximity, or datacenter health.

**Implementation**: DNS-based. When a user queries `api.example.com`, the authoritative DNS server (Route 53, Cloudflare) checks:
1. User's IP geolocation or latency measurements.
2. Health of each datacenter.
3. Returns the IP of the closest healthy datacenter.

**Failover**: If `us-east-1` is down, the health check from Route 53 detects this, and new DNS responses point to `eu-west-1`. Existing connections to the dead DC break, but new ones go to the healthy one.

**TTL considerations**: Short TTLs (30–60 seconds) for fast failover, but increase DNS query load. Long TTLs (5 minutes) reduce DNS load but slow failover.

Real-world: Cloudflare Anycast routes all `1.1.1.1` queries to the nearest PoP using BGP routing, not DNS, which is even faster.

---

**Q: How do you make a load balancer itself highly available?**

A single LB is a single point of failure. Solutions:

**Active-Passive with VIP failover (keepalived/VRRP)**:
- Two LB nodes share a virtual IP.
- The active node holds the VIP and processes traffic.
- The passive node sends heartbeats and takes over the VIP if the active node dies (failover in ~1–3 seconds).
- Used by on-prem HAProxy deployments.

**Active-Active with ECMP**:
- Multiple LB nodes share traffic via Equal-Cost Multi-Path routing.
- Routers distribute packets across all LBs.
- If one LB dies, the router redistributes.
- Problem: TCP connections are hashed per-flow; an LB dying breaks existing connections.

**Cloud-managed LBs**:
- AWS ALB/NLB are fully managed and internally HA across AZs.
- You don't manage the LB instances yourself.

**Connection draining** (graceful shutdown):
When removing an LB or backend from rotation, send a signal to stop accepting new connections but allow existing ones to finish (typically a 30–300 second drain period). AWS Target Groups support this natively.

---

**Q: What are the performance limits of a software load balancer like Nginx or HAProxy?**

| Component | Approximate limit |
|---|---|
| Nginx (commodity server, 32 cores) | ~1 million req/sec HTTP |
| HAProxy | ~500k–2M connections/sec depending on mode |
| AWS NLB | Scales to millions of req/sec automatically |
| L4 kernel bypass (DPDK-based) | 10–100M pps |

Bottlenecks:
- **CPU**: TLS termination is expensive (~10–20% CPU per core at high rates). Offload with hardware SSL accelerators or use session resumption (TLS 1.3 0-RTT).
- **Memory**: Each active connection holds ~4–20KB of state. 1M connections = 4–20GB RAM.
- **Network bandwidth**: A 10Gbps NIC limits throughput to ~1.2 GB/s regardless of CPU.

At extreme scale (Google, Facebook), they use kernel-bypass networking (DPDK, XDP/eBPF) to process packets at line rate without context switches.
