# System Design Fundamentals

> **Author:** Aniket Waichal

A structured collection of in-depth Q&A notes covering core system design concepts.
Built for engineers preparing for system design interviews or strengthening their
foundational knowledge. Each answer covers the concept, trade-offs, real-world
examples, and numbers where relevant.

---

## Repository Structure

```
System_design/
│
├── README.md
│
├── foundations/                        # Core theory every engineer must know first
│   ├── 01-scalability.md
│   ├── 02-latency-throughput.md
│   ├── 03-availability-reliability.md
│   ├── 04-consistency-models.md
│   ├── 05-cap-theorem.md
│   ├── 06-networking-basics.md
│   └── 07-back-of-envelope.md
│
└── components/
    ├── distributed-systems/            # Individual building blocks
    │   ├── 08-caching.md
    │   ├── 09-databases.md
    │   ├── 10-message-queues.md
    │   ├── 11-load-balancers.md
    │   ├── 12-proxies-and-api-gateways.md
    │   ├── 13-storage.md
    │   ├── 14-rate-limiting.md
    │   └── 15-api-design.md
    │
    └── case-studies/                   # End-to-end real-world system designs
        ├── url-shortener.md
        ├── twitter-feed.md
        ├── whatsapp.md
        ├── youtube.md
        └── uber.md
```

---

## Contents

### Foundations

Start here. These are the vocabulary and mental models that every other topic builds on.

| # | File | What's covered |
|---|------|----------------|
| 01 | [01-scalability.md](./foundations/01-scalability.md) | Vertical vs horizontal scaling, load balancing, sharding, stateless design |
| 02 | [02-latency-throughput.md](./foundations/02-latency-throughput.md) | Latency, throughput, percentiles (P50/P99/P999), bottlenecks, queueing theory |
| 03 | [03-availability-reliability.md](./foundations/03-availability-reliability.md) | The nines, SLA/SLO/SLI, error budgets, fault tolerance, cascading failures |
| 04 | [04-consistency-models.md](./foundations/04-consistency-models.md) | Strong, eventual, causal consistency, read-your-writes, Cassandra levels |
| 05 | [05-cap-theorem.md](./foundations/05-cap-theorem.md) | CAP theorem, CP vs AP, PACELC, Google Spanner, split-brain, tunable consistency |
| 06 | [06-networking-basics.md](./foundations/06-networking-basics.md) | OSI model, TCP/UDP, HTTP/1.1/2/3, TLS, DNS, CDN, WebSockets |
| 07 | [07-back-of-envelope.md](./foundations/07-back-of-envelope.md) | QPS, storage, bandwidth estimation, worked examples for WhatsApp and YouTube |

### Components — Distributed Systems

Individual building blocks. Read these after foundations; they assume familiarity with
consistency, replication, and networking.

| # | File | What's covered |
|---|------|----------------|
| 08 | [08-caching.md](./components/distributed%20systems/08-caching.md) | Cache-aside, write-through, write-back, eviction (LRU/LFU), Redis vs Memcached |
| 09 | [09-databases.md](./components/distributed%20systems/09-databases.md) | SQL vs NoSQL, ACID, BASE, indexing (B-tree/LSM), replication, N+1 problem |
| 10 | [10-message-queues.md](./components/distributed%20systems/10-message-queues.md) | Kafka, RabbitMQ, pub/sub, at-least-once vs exactly-once delivery |
| 11 | [11-load-balancers.md](./components/distributed%20systems/11-load-balancers.md) | L4 vs L7, algorithms, sticky sessions, health checks, GSLB, HA |
| 12 | [12-proxies-and-api-gateways.md](./components/distributed%20systems/12-proxies-and-api-gateways.md) | Forward/reverse proxies, API gateway patterns, service mesh |
| 13 | [13-storage.md](./components/distributed%20systems/13-storage.md) | Block, object, file storage; S3 architecture, HDFS, replication strategies |
| 14 | [14-rate-limiting.md](./components/distributed%20systems/14-rate-limiting.md) | Token bucket, leaky bucket, sliding window, distributed rate limiting with Redis |
| 15 | [15-api-design.md](./components/distributed%20systems/15-api-design.md) | REST vs GraphQL vs gRPC, idempotency, versioning, pagination patterns |

### Components — Case Studies

End-to-end system designs. Each one is a full interview-depth walkthrough: requirements,
back-of-envelope estimation, data model, core architecture, and key trade-offs.

| File | System | Core challenge |
|------|--------|----------------|
| [url-shortener.md](./components/case-studies/url-shortener.md) | URL Shortener | ID generation, 115k redirect req/sec, multi-layer caching |
| [twitter-feed.md](./components/case-studies/twitter-feed.md) | Twitter Feed | Fan-out on write vs read, celebrity problem, Redis sorted set timelines |
| [whatsapp.md](./components/case-studies/whatsapp.md) | WhatsApp | 167M concurrent WebSockets, guaranteed delivery, Signal Protocol E2EE |
| [youtube.md](./components/case-studies/youtube.md) | YouTube | Transcoding pipeline, adaptive bitrate streaming, 167 Tbps CDN |
| [uber.md](./components/case-studies/uber.md) | Uber | 500k location writes/sec, geohash queries, distributed matching lock |

---

## How to Use This Repo

Each file covers one topic in question-and-answer format. Answers are written to be
detailed enough for a senior-level system design interview — concept, trade-offs,
real-world examples, and numbers.

**Suggested study path:**

1. Work through `foundations/` in order (01 → 07). These build on each other.
2. Read `components/distributed systems/` in any order — pick topics relevant to
   interviews you have coming up.
3. Tackle `components/case-studies/` last. Each case study pulls together multiple
   component topics; they'll make more sense once the building blocks are solid.
4. **Test yourself:** Re-read only the question, write or say your answer out loud,
   then check. Focus on trade-offs — interviewers want "it depends, and here is why",
   not textbook definitions.

---

## Roadmap

**Foundations (planned)**
- `consistent-hashing.md` — Virtual nodes, ring topology, used in Cassandra and DynamoDB

**Case Studies**
- ✅ `url-shortener.md`
- ✅ `twitter-feed.md`
- ✅ `whatsapp.md`
- ✅ `youtube.md`
- ✅ `uber.md`
- `netflix.md` — Video streaming, content delivery, chaos engineering
- `google-maps.md` — Geo-indexing, routing graphs, tile serving
- `notion.md` — Real-time collaborative editing, CRDT/OT

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