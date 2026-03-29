# System Design Fundamentals

> **Author:** Aniket Waichal

A structured collection of in-depth Q&A notes covering core system design concepts.
Built for engineers preparing for system design interviews or strengthening their
foundational knowledge.

---

## Contents

| File | Topic |
|------|-------|
| [01-scalability.md](./01-scalability.md) | Vertical vs horizontal scaling, load balancers, sharding, stateless design |
| [02-latency-throughput.md](./02-latency-throughput.md) | Latency, throughput, percentiles, P99, bottlenecks, queueing theory |
| [03-availability-reliability.md](./03-availability-reliability.md) | Nines, SLA/SLO/SLI, error budgets, fault tolerance, cascading failures |
| [04-consistency-models.md](./04-consistency-models.md) | Strong, eventual, causal consistency, read-your-writes, Cassandra levels |
| [05-cap-theorem.md](./05-cap-theorem.md) | CAP theorem, CP vs AP systems, PACELC, real-world database choices |
| [06-networking-basics.md](./06-networking-basics.md) | OSI model, TCP/UDP, HTTP/1.1/2/3, TLS, DNS, CDN, WebSockets |
| [07-back-of-envelope.md](./07-back-of-envelope.md) | QPS, storage, bandwidth estimation, WhatsApp/YouTube worked examples |

---

## How to Use This Repo

Each file covers one topic in question and answer format. Every answer is written
to be detailed enough to use directly in a system design interview — covering the
concept, the trade-offs, real-world examples, and the numbers where relevant.

Suggested approach:
1. Read through a file once to build familiarity.
2. Come back and test yourself — read only the question, write or say your answer,
   then check against the written answer.
3. Focus on trade-offs, not just definitions. Interviewers want to hear "it depends,
   and here is why" not just textbook definitions.

---

## Topics Covered

- Scalability — vertical vs horizontal scaling, load balancing algorithms, auto-scaling,
  database sharding, the N+1 problem
- Latency & Throughput — percentile latencies (P50/P99/P999), Little's Law, tail latency
  amplification, queueing theory, bottleneck identification
- Availability & Reliability — the nines table, MTBF/MTTR, SLA vs SLO vs SLI, error budgets,
  circuit breakers, cascading failures, chaos engineering
- Consistency Models — strong/linearizable, eventual, causal, read-your-writes, monotonic
  reads, Cassandra consistency levels, ACID vs CAP consistency
- CAP Theorem — why P is non-negotiable, CP vs AP behavior during partitions, PACELC theorem,
  Google Spanner, dynamic consistency tuning
- Networking — OSI layers, TCP 3-way handshake, congestion control, UDP use cases,
  HTTP/1.1 vs HTTP/2 vs HTTP/3, TLS 1.3 handshake, full DNS resolution walkthrough,
  CDN internals, reverse proxies, WebSockets
- Back-of-Envelope Estimation — key numbers to memorize, QPS/storage/bandwidth/server count
  formulas, full worked examples for WhatsApp and YouTube, common estimation mistakes

---

## Who This Is For

- Engineers preparing for system design interviews at any level
- Developers moving from individual services to distributed systems thinking
- Anyone who wants a concise but deep reference for these foundational concepts

---

## Roadmap

Planned future files:

- `08-caching.md` — Cache aside, write-through, write-back, eviction policies, Redis internals
- `09-databases.md` — SQL vs NoSQL, indexing, B-trees, LSM trees, ACID, transactions
- `10-message-queues.md` — Kafka, RabbitMQ, pub/sub, at-least-once vs exactly-once delivery
- `11-consistent-hashing.md` — Virtual nodes, ring topology, used in Cassandra and DynamoDB
- `12-rate-limiting.md` — Token bucket, leaky bucket, sliding window, distributed rate limiting
- `13-api-design.md` — REST vs GraphQL vs gRPC, idempotency, versioning, pagination
- `14-real-world-systems.md` — Design URL shortener, Twitter feed, WhatsApp, YouTube, Uber

---

## Contributing

If you spot an error, an outdated fact, or a concept that deserves a deeper answer,
open an issue or a pull request.

---

## License

MIT — use freely for personal study or sharing with your team.