# System Design Fundamentals

> **Author:** Aniket Waichal

A structured collection of in-depth Q&A notes covering core system design concepts.
Built for engineers preparing for system design interviews or strengthening their
foundational knowledge.

---

## Recommended Folder Structure

As the repo grows, keep files organised into three folders:

```
System_design/
│
├── README.md
│
├── foundations/               # Core concepts every engineer must know
│   ├── 01-scalability.md
│   ├── 02-latency-throughput.md
│   ├── 03-availability-reliability.md
│   ├── 04-consistency-models.md
│   ├── 05-cap-theorem.md
│   ├── 06-networking-basics.md
│   └── 07-back-of-envelope.md
│
├── components/                # Individual building blocks of distributed systems
│   ├── 08-caching.md
│   ├── 09-databases.md
│   ├── 10-message-queues.md
│   ├── 11-load-balancers.md
│   ├── 12-proxies-and-api-gateways.md
│   ├── 13-storage.md
│   ├── 14-rate-limiting.md
│   ├── 15-api-design.md
│   └── 16-consistent-hashing.md
│
└── case-studies/              # End-to-end real-world system designs
    ├── url-shortener.md
    ├── twitter-feed.md
    ├── whatsapp.md
    ├── youtube.md
    └── uber.md
```

> **File naming convention:** use lowercase kebab-case with a numeric prefix
> (`01-scalability.md`) so files sort correctly in any file browser or IDE.
> Keep it consistent — currently the repo mixes styles
> (`load_balancers.md`, `api_design.md`, `networking-basics.md`).

---

## Current Contents

### Foundations

| # | File | Topic |
|---|------|-------|
| 01 | [scalability.md](./scalability.md) | Vertical vs horizontal scaling, load balancers, sharding, stateless design |
| 02 | [latency-throughput.md](./latency-throughput.md) | Latency, throughput, percentiles, P99, bottlenecks, queueing theory |
| 03 | [availability-reliability.md](./availability-reliability.md) | Nines, SLA/SLO/SLI, error budgets, fault tolerance, cascading failures |
| 04 | [consistency-models.md](./consistency-models.md) | Strong, eventual, causal consistency, read-your-writes, Cassandra levels |
| 05 | [networking-basics.md](./networking-basics.md) | OSI model, TCP/UDP, HTTP/1.1/2/3, TLS, DNS, CDN, WebSockets |
| 06 | [back-of-envelope.md](./back-of-envelope.md) | QPS, storage, bandwidth estimation, worked examples |

### Components

| # | File | Topic |
|---|------|-------|
| 07 | [load_balancers.md](./load_balancers.md) | Algorithms, L4 vs L7, health checks, sticky sessions |
| 08 | [caching.md](./caching.md) | Cache-aside, write-through, write-back, eviction, Redis internals |
| 09 | [databases.md](./databases.md) | SQL vs NoSQL, indexing, B-trees, LSM trees, ACID, transactions |
| 10 | [message_queues.md](./message_queues.md) | Kafka, RabbitMQ, pub/sub, at-least-once vs exactly-once delivery |
| 11 | [storage.md](./storage.md) | Block, object, file storage; S3, HDFS, replication |
| 12 | [proxies_and_api_gateways.md](./proxies_and_api_gateways.md) | Forward/reverse proxies, API gateway patterns |
| 13 | [rate_limiting.md](./rate_limiting.md) | Token bucket, leaky bucket, sliding window, distributed rate limiting |
| 14 | [api_design.md](./api_design.md) | REST vs GraphQL vs gRPC, idempotency, versioning, pagination |

---

## How to Use This Repo

Each file covers one topic in question-and-answer format. Every answer is written
to be detailed enough for a system design interview — covering the concept,
the trade-offs, real-world examples, and numbers where relevant.

**Suggested study approach:**

1. **Read** through a file once to build familiarity.
2. **Test yourself** — read only the question, write or say your answer, then check.
3. **Focus on trade-offs.** Interviewers want "it depends, and here is why",
   not textbook definitions.
4. **Use the structure** — start with `foundations/` before jumping into `components/`.
   The case studies make most sense once both are solid.

---

## Topics Covered

**Foundations**
- Scalability — vertical vs horizontal scaling, load balancing algorithms, auto-scaling, database sharding, the N+1 problem
- Latency & Throughput — percentile latencies (P50/P99/P999), Little's Law, tail latency amplification, queueing theory, bottleneck identification
- Availability & Reliability — the nines table, MTBF/MTTR, SLA vs SLO vs SLI, error budgets, circuit breakers, cascading failures, chaos engineering
- Consistency Models — strong/linearizable, eventual, causal, read-your-writes, monotonic reads, Cassandra consistency levels, ACID vs CAP consistency
- Networking — OSI layers, TCP 3-way handshake, congestion control, UDP use cases, HTTP/1.1 vs HTTP/2 vs HTTP/3, TLS 1.3 handshake, full DNS resolution, CDN internals, WebSockets
- Back-of-Envelope Estimation — key numbers to memorise, QPS/storage/bandwidth/server count formulas, worked examples for WhatsApp and YouTube

**Components**
- Load Balancers — L4 vs L7, round-robin, least-connections, consistent hashing, health checks, sticky sessions
- Caching — cache-aside, write-through, write-back, eviction policies (LRU/LFU), Redis vs Memcached internals
- Databases — SQL vs NoSQL trade-offs, B-tree and LSM-tree indexes, ACID transactions, replication, sharding
- Message Queues — Kafka internals, RabbitMQ, pub/sub patterns, at-least-once vs exactly-once semantics
- Storage — block, object, and file storage; S3 architecture, HDFS, replication strategies
- Proxies & API Gateways — forward vs reverse proxy, API gateway responsibilities, service mesh
- Rate Limiting — token bucket, leaky bucket, fixed and sliding window, distributed rate limiting with Redis
- API Design — REST constraints, GraphQL trade-offs, gRPC and Protobuf, idempotency keys, pagination patterns

---

## Roadmap

**Foundations (planned)**
- `cap-theorem.md` — CAP theorem, CP vs AP systems, PACELC, real-world database choices

**Components (planned)**
- `consistent-hashing.md` — Virtual nodes, ring topology, Cassandra and DynamoDB usage

**Case Studies (planned)**
- `url-shortener.md` — Hash functions, collision handling, redirect strategies
- `twitter-feed.md` — Fan-out on write vs read, timeline generation, celebrity problem
- `whatsapp.md` — Message delivery guarantees, presence, end-to-end encryption at scale
- `youtube.md` — Video ingestion, transcoding pipeline, CDN distribution
- `uber.md` — Geo-indexing, real-time matching, surge pricing

---

## Who This Is For

- Engineers preparing for system design interviews at any level
- Developers moving from individual services to distributed-systems thinking
- Anyone who wants a concise but deep reference for these foundational concepts

---

## Contributing

If you spot an error, an outdated fact, or a concept that deserves a deeper answer,
open an issue or a pull request.

---

## License

MIT — use freely for personal study or sharing with your team.