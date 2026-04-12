# 🏗️ System Design — Complete Knowledge Base

<p align="center">
  <b>Author:</b> Aniket Waichal<br>
  <b>A Structured, Interview-Depth System Design Knowledge Base</b><br>
  Built for engineers who want to go beyond definitions — covering concepts, trade-offs, real-world examples, and numbers.
</p>

---

<p align="center">
  <img src="https://img.shields.io/badge/HLD_Topics-57-blue" />
  <img src="https://img.shields.io/badge/LLD_Topics-64-orange" />
  <img src="https://img.shields.io/badge/Case_Studies-16-green" />
  <img src="https://img.shields.io/badge/Format-Q%26A%20Notes-purple" />
  <img src="https://img.shields.io/badge/License-MIT-lightgrey" />
</p>

---

## 📁 Complete Repository Structure

```
System_design/
│
├── README.md
│
├── hld/                                                    # High Level Design
│   │
│   ├── foundations/                                        # ✅ COMPLETED
│   │   ├── 01-scalability.md                               ✅
│   │   ├── 02-latency-throughput.md                        ✅
│   │   ├── 03-availability-reliability.md                  ✅
│   │   ├── 04-consistency-models.md                        ✅
│   │   ├── 05-cap-theorem.md                               ✅
│   │   ├── 06-networking-basics.md                         ✅
│   │   └── 07-back-of-envelope.md                          ✅
│   │
│   ├── components/                                         # ✅ COMPLETED
│   │   ├── 08-caching.md                                   ✅
│   │   ├── 09-databases.md                                 ✅
│   │   ├── 10-message-queues.md                            ✅
│   │   ├── 11-load-balancers.md                            ✅
│   │   ├── 12-proxies-and-api-gateways.md                  ✅
│   │   ├── 13-storage.md                                   ✅
│   │   ├── 14-rate-limiting.md                             ✅
│   │   └── 15-api-design.md                                ✅
│   │
│   ├── distributed-systems/                                # ✅ COMPLETED
│   │   ├── 16-replication.md                               ✅
│   │   ├── 17-sharding-and-partitioning.md                 ✅
│   │   ├── 18-consistent-hashing.md                        ✅
│   │   ├── 19-consensus-algorithms.md                      ✅
│   │   ├── 20-distributed-transactions.md                  ✅
│   │   ├── 21-event-driven-architecture.md                 ✅
│   │   ├── 22-service-discovery.md                         ✅
│   │   ├── 23-service-mesh.md                              ✅
│   │   ├── 24-distributed-locking.md                       ✅
│   │   ├── 25-bloom-filters.md                             ✅
│   │   ├── 26-failure-detection.md                         ✅
│   │   ├── 27-leader-election.md                           ✅
│   │   ├── 28-gossip-protocol.md                           ✅
│   │   └── 29-quorum.md                                    ✅
│   │
│   ├── observability/                                      # ✅ COMPLETED
│   │   ├── 30-distributed-tracing.md                       ✅
│   │   ├── 31-logging-at-scale.md                          ✅
│   │   ├── 32-metrics-and-monitoring.md                    ✅
│   │   ├── 33-alerting-strategies.md                       ✅
│   │   ├── 34-circuit-breaker.md                           ✅
│   │   ├── 35-bulkhead-pattern.md                          ✅
│   │   ├── 36-retry-and-backoff.md                         ✅
│   │   └── 37-chaos-engineering.md                         ✅
│   │
│   ├── security/                                           # ✅ COMPLETED
│   │   ├── 38-authentication-and-authorization.md          ✅
│   │   ├── 39-https-tls-mtls.md                            ✅
│   │   ├── 40-secrets-management.md                        ✅
│   │   └── 41-ddos-protection.md                           ✅
│   │
│   └── case-studies/                                       # ✅ 5 done — 11 remaining
│       ├── 42-url-shortener.md                             ✅
│       ├── 43-whatsapp.md                                  ✅
│       ├── 44-twitter-feed.md                              ✅
│       ├── 45-uber.md                                      ✅
│       ├── 46-youtube.md                                   ✅
│       ├── 47-google-drive.md                              ⬜
│       ├── 48-search-autocomplete.md                       ⬜
│       ├── 49-notification-system.md                       ⬜
│       ├── 50-web-crawler.md                               ⬜
│       ├── 51-pastebin.md                                  ⬜
│       ├── 52-distributed-job-scheduler.md                 ⬜
│       ├── 53-payment-system.md                            ⬜
│       ├── 54-flash-sale-system.md                         ⬜
│       ├── 55-google-maps.md                               ⬜
│       ├── 56-netflix-recommendation.md                    ⬜
│       └── 57-rag-llm-infrastructure.md                    ⬜
│
└── lld/                                                    # Low Level Design
    │
    ├── oop-fundamentals/                                   # ✅ COMPLETED
    │   ├── 58-classes-and-objects.md                       ✅
    │   ├── 59-encapsulation.md                             ✅
    │   ├── 60-abstraction.md                               ✅
    │   ├── 61-inheritance.md                               ✅
    │   ├── 62-polymorphism.md                              ✅
    │   ├── 63-interfaces-vs-abstract-classes.md            ✅
    │   └── 64-composition-over-inheritance.md              ✅
    │
    ├── solid-principles/                                   # ✅ COMPLETED
    │   ├── 65-single-responsibility.md                     ✅
    │   ├── 66-open-closed.md                               ✅
    │   ├── 67-liskov-substitution.md                       ✅
    │   ├── 68-interface-segregation.md                     ✅
    │   └── 69-dependency-inversion.md                      ✅
    │
    ├── design-patterns/
    │   ├── creational/                                     # ✅ COMPLETED
    │   │   ├── 70-singleton.md                             ✅
    │   │   ├── 71-factory-method.md                        ✅
    │   │   ├── 72-abstract-factory.md                      ✅
    │   │   ├── 73-builder.md                               ✅
    │   │   └── 74-prototype.md                             ✅
    │   │
    │   ├── structural/                                     # ⬜ TODO
    │   │   ├── 75-adapter.md                               ⬜
    │   │   ├── 76-decorator.md                             ⬜
    │   │   ├── 77-facade.md                                ⬜
    │   │   ├── 78-proxy.md                                 ⬜
    │   │   ├── 79-composite.md                             ⬜
    │   │   ├── 80-bridge.md                                ⬜
    │   │   └── 81-flyweight.md                             ⬜
    │   │
    │   └── behavioral/                                     # ⬜ TODO
    │       ├── 82-observer.md                              ⬜
    │       ├── 83-strategy.md                              ⬜
    │       ├── 84-chain-of-responsibility.md               ⬜
    │       ├── 85-command.md                               ⬜
    │       ├── 86-iterator.md                              ⬜
    │       ├── 87-template-method.md                       ⬜
    │       ├── 88-state.md                                 ⬜
    │       ├── 89-mediator.md                              ⬜
    │       └── 90-memento.md                               ⬜
    │
    ├── uml-and-diagrams/                                   # ⬜ TODO
    │   ├── 91-class-diagrams.md                            ⬜
    │   ├── 92-sequence-diagrams.md                         ⬜
    │   ├── 93-use-case-diagrams.md                         ⬜
    │   └── 94-er-diagrams.md                               ⬜
    │
    ├── database-schema-design/                             # ⬜ TODO
    │   ├── 95-normalization.md                             ⬜
    │   ├── 96-keys-and-constraints.md                      ⬜
    │   ├── 97-indexing-strategies.md                       ⬜
    │   ├── 98-relationships.md                             ⬜
    │   ├── 99-junction-tables.md                           ⬜
    │   ├── 100-soft-delete-and-timestamps.md               ⬜
    │   └── 101-uuid-vs-autoincrement.md                    ⬜
    │
    ├── api-contract-design/                                # ⬜ TODO
    │   ├── 102-restful-principles.md                       ⬜
    │   ├── 103-http-status-codes.md                        ⬜
    │   ├── 104-request-response-schema.md                  ⬜
    │   ├── 105-versioning-strategies.md                    ⬜
    │   ├── 106-pagination.md                               ⬜
    │   ├── 107-idempotency.md                              ⬜
    │   └── 108-error-response-standards.md                 ⬜
    │
    └── practice-problems/                                  # ⬜ TODO
        ├── beginner/
        │   ├── 109-parking-lot.md                          ⬜
        │   ├── 110-library-management.md                   ⬜
        │   ├── 111-atm.md                                  ⬜
        │   ├── 112-vending-machine.md                      ⬜
        │   └── 113-chess-game.md                           ⬜
        ├── intermediate/
        │   ├── 114-hotel-booking.md                        ⬜
        │   ├── 115-elevator-system.md                      ⬜
        │   ├── 116-food-delivery-app.md                    ⬜
        │   ├── 117-cab-booking.md                          ⬜
        │   └── 118-movie-ticket-booking.md                 ⬜
        └── advanced/
            ├── 119-notification-service.md                 ⬜
            ├── 120-lru-cache.md                            ⬜
            └── 121-llm-rag-pipeline.md                     ⬜
```

---

## 📊 Progress Tracker

| Section | Total | Done | Remaining |
|---|---|---|---|
| HLD — Foundations | 7 | ✅ 7 | 0 |
| HLD — Components | 8 | ✅ 8 | 0 |
| HLD — Distributed Systems Deep Dive | 14 | ✅ 14 | 0 |
| HLD — Observability & Reliability | 8 | ✅ 8 | 0 |
| HLD — Security | 4 | ✅ 4 | 0 |
| HLD — Case Studies | 16 | ✅ 5 | ⬜ 11 |
| LLD — OOP Fundamentals | 7 | ✅ 7 | 0 |
| LLD — SOLID Principles | 5 | ✅ 5 | 0 |
| LLD — Design Patterns — Creational | 5 | ✅ 5 | 0 |
| LLD — Design Patterns — Structural | 7 | 0 | ⬜ 7 |
| LLD — Design Patterns — Behavioral | 9 | 0 | ⬜ 9 |
| LLD — UML & Diagrams | 4 | 0 | ⬜ 4 |
| LLD — Database Schema Design | 7 | 0 | ⬜ 7 |
| LLD — API Contract Design | 7 | 0 | ⬜ 7 |
| LLD — Practice Problems | 13 | 0 | ⬜ 13 |
| **TOTAL** | **121** | **✅ 63** | **⬜ 58** |

---

## 📚 Contents

---

### 🟣 HLD — HIGH LEVEL DESIGN

---

#### 🧱 Foundations
> Start here. These are the vocabulary and mental models that every other topic builds on.

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 01 | [01-scalability.md](./hld/foundations/01-scalability.md) | Vertical vs horizontal scaling, load balancing, sharding, stateless design | ✅ |
| 02 | [02-latency-throughput.md](./hld/foundations/02-latency-throughput.md) | Latency, throughput, percentiles (P50/P99/P999), bottlenecks, queueing theory | ✅ |
| 03 | [03-availability-reliability.md](./hld/foundations/03-availability-reliability.md) | The nines, SLA/SLO/SLI, error budgets, fault tolerance, cascading failures | ✅ |
| 04 | [04-consistency-models.md](./hld/foundations/04-consistency-models.md) | Strong, eventual, causal consistency, read-your-writes, Cassandra levels | ✅ |
| 05 | [05-cap-theorem.md](./hld/foundations/05-cap-theorem.md) | CAP theorem, CP vs AP, PACELC, Google Spanner, split-brain | ✅ |
| 06 | [06-networking-basics.md](./hld/foundations/06-networking-basics.md) | OSI model, TCP/UDP, HTTP/1.1/2/3, TLS, DNS, CDN, WebSockets | ✅ |
| 07 | [07-back-of-envelope.md](./hld/foundations/07-back-of-envelope.md) | QPS, storage, bandwidth estimation, worked examples for WhatsApp and YouTube | ✅ |

---

#### ⚙️ Components
> Individual building blocks. Read these after foundations.

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 08 | [08-caching.md](./hld/components/08-caching.md) | Cache-aside, write-through, write-back, eviction (LRU/LFU), Redis vs Memcached | ✅ |
| 09 | [09-databases.md](./hld/components/09-databases.md) | SQL vs NoSQL, ACID, BASE, indexing (B-tree/LSM), replication, N+1 problem | ✅ |
| 10 | [10-message-queues.md](./hld/components/10-message-queues.md) | Kafka, RabbitMQ, pub/sub, at-least-once vs exactly-once delivery | ✅ |
| 11 | [11-load-balancers.md](./hld/components/11-load-balancers.md) | L4 vs L7, algorithms, sticky sessions, health checks, GSLB, HA | ✅ |
| 12 | [12-proxies-and-api-gateways.md](./hld/components/12-proxies-and-api-gateways.md) | Forward/reverse proxies, API gateway patterns, service mesh | ✅ |
| 13 | [13-storage.md](./hld/components/13-storage.md) | Block, object, file storage; S3 architecture, HDFS, replication strategies | ✅ |
| 14 | [14-rate-limiting.md](./hld/components/14-rate-limiting.md) | Token bucket, leaky bucket, sliding window, distributed rate limiting with Redis | ✅ |
| 15 | [15-api-design.md](./hld/components/15-api-design.md) | REST vs GraphQL vs gRPC, idempotency, versioning, pagination patterns | ✅ |

---

#### 🔬 Distributed Systems — Deep Dive
> How systems behave at scale across multiple machines.

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 16 | [16-replication.md](./hld/distributed-systems/16-replication.md) | Leader-follower, multi-leader, leaderless (Dynamo-style), replication lag | ✅ |
| 17 | [17-sharding-and-partitioning.md](./hld/distributed-systems/17-sharding-and-partitioning.md) | Hash, range, directory-based sharding, hotspot problems | ✅ |
| 18 | [18-consistent-hashing.md](./hld/distributed-systems/18-consistent-hashing.md) | Virtual nodes, ring topology, Cassandra and DynamoDB usage | ✅ |
| 19 | [19-consensus-algorithms.md](./hld/distributed-systems/19-consensus-algorithms.md) | Raft, Paxos (conceptual), split-brain, leader election in consensus | ✅ |
| 20 | [20-distributed-transactions.md](./hld/distributed-systems/20-distributed-transactions.md) | 2-phase commit, 3-phase commit, sagas (choreography vs orchestration) | ✅ |
| 21 | [21-event-driven-architecture.md](./hld/distributed-systems/21-event-driven-architecture.md) | Event sourcing, CQRS, outbox pattern | ✅ |
| 22 | [22-service-discovery.md](./hld/distributed-systems/22-service-discovery.md) | Client-side, server-side, Zookeeper, Consul, Eureka | ✅ |
| 23 | [23-service-mesh.md](./hld/distributed-systems/23-service-mesh.md) | Sidecar pattern, Istio, Envoy, east-west traffic | ✅ |
| 24 | [24-distributed-locking.md](./hld/distributed-systems/24-distributed-locking.md) | Redis Redlock, Zookeeper-based locking, fencing tokens | ✅ |
| 25 | [25-bloom-filters.md](./hld/distributed-systems/25-bloom-filters.md) | Probabilistic data structures, false positive rates, use cases | ✅ |
| 26 | [26-failure-detection.md](./hld/distributed-systems/26-failure-detection.md) | Heartbeats, timeouts, phi accrual failure detector | ✅ |
| 27 | [27-leader-election.md](./hld/distributed-systems/27-leader-election.md) | Bully algorithm, ring algorithm, Zookeeper-based election | ✅ |
| 28 | [28-gossip-protocol.md](./hld/distributed-systems/28-gossip-protocol.md) | Epidemic dissemination, membership, Cassandra usage | ✅ |
| 29 | [29-quorum.md](./hld/distributed-systems/29-quorum.md) | Read/write quorum, W+R>N, Dynamo-style tunable consistency | ✅ |

---

#### 🔭 Observability & Reliability

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 30 | [30-distributed-tracing.md](./hld/observability/30-distributed-tracing.md) | OpenTelemetry, Jaeger, Zipkin, trace IDs, span propagation | ✅ |
| 31 | [31-logging-at-scale.md](./hld/observability/31-logging-at-scale.md) | Structured logging, log aggregation, ELK stack, Loki | ✅ |
| 32 | [32-metrics-and-monitoring.md](./hld/observability/32-metrics-and-monitoring.md) | Prometheus, Grafana, Datadog, RED method, USE method | ✅ |
| 33 | [33-alerting-strategies.md](./hld/observability/33-alerting-strategies.md) | Alert fatigue, SLO-based alerting, runbooks | ✅ |
| 34 | [34-circuit-breaker.md](./hld/observability/34-circuit-breaker.md) | Closed/open/half-open states, Hystrix, Resilience4j | ✅ |
| 35 | [35-bulkhead-pattern.md](./hld/observability/35-bulkhead-pattern.md) | Isolating failures, thread pools, semaphores | ✅ |
| 36 | [36-retry-and-backoff.md](./hld/observability/36-retry-and-backoff.md) | Exponential backoff, jitter, retry storms, idempotency | ✅ |
| 37 | [37-chaos-engineering.md](./hld/observability/37-chaos-engineering.md) | Chaos Monkey, game days, blast radius, hypothesis-driven testing | ✅ |

---

#### 🔐 Security

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 38 | [38-authentication-and-authorization.md](./hld/security/38-authentication-and-authorization.md) | OAuth2, JWT, API keys, RBAC, ABAC | ✅ |
| 39 | [39-https-tls-mtls.md](./hld/security/39-https-tls-mtls.md) | TLS handshake, certificate pinning, mutual TLS in microservices | ✅ |
| 40 | [40-secrets-management.md](./hld/security/40-secrets-management.md) | Vault, AWS Secrets Manager, rotation, injection patterns | ✅ |
| 41 | [41-ddos-protection.md](./hld/security/41-ddos-protection.md) | Rate limiting, WAF, Cloudflare, volumetric vs application layer attacks | ✅ |

---

#### 🔍 Case Studies

| # | File | Core Challenge | Status |
|---|------|----------------|--------|
| 42 | [42-url-shortener.md](./hld/case-studies/42-url-shortener.md) | ID generation, 115k redirect req/sec, multi-layer caching | ✅ |
| 43 | [43-whatsapp.md](./hld/case-studies/43-whatsapp.md) | 167M concurrent WebSockets, guaranteed delivery, Signal Protocol E2EE | ✅ |
| 44 | [44-twitter-feed.md](./hld/case-studies/44-twitter-feed.md) | Fan-out on write vs read, celebrity problem, Redis sorted sets | ✅ |
| 45 | [45-uber.md](./hld/case-studies/45-uber.md) | 500k location writes/sec, geohash queries, distributed matching lock | ✅ |
| 46 | [46-youtube.md](./hld/case-studies/46-youtube.md) | Transcoding pipeline, adaptive bitrate streaming, 167 Tbps CDN | ✅ |
| 47 | [47-google-drive.md](./hld/case-studies/47-google-drive.md) | File chunking, delta sync, conflict resolution, offline support | ⬜ |
| 48 | [48-search-autocomplete.md](./hld/case-studies/48-search-autocomplete.md) | Trie, top-K with heap, distributed trie, prefix caching | ⬜ |
| 49 | [49-notification-system.md](./hld/case-studies/49-notification-system.md) | Push/email/SMS, fan-out, priority queues, deduplication | ⬜ |
| 50 | [50-web-crawler.md](./hld/case-studies/50-web-crawler.md) | BFS/DFS, politeness, deduplication with bloom filter, DNS caching | ⬜ |
| 51 | [51-pastebin.md](./hld/case-studies/51-pastebin.md) | Object storage, expiry, unique key generation, read-heavy caching | ⬜ |
| 52 | [52-distributed-job-scheduler.md](./hld/case-studies/52-distributed-job-scheduler.md) | Cron at scale, idempotency, exactly-once execution, failure recovery | ⬜ |
| 53 | [53-payment-system.md](./hld/case-studies/53-payment-system.md) | Exactly-once semantics, idempotency keys, double-spend prevention | ⬜ |
| 54 | [54-flash-sale-system.md](./hld/case-studies/54-flash-sale-system.md) | Inventory locking, queue-based ordering, cache stampede prevention | ⬜ |
| 55 | [55-google-maps.md](./hld/case-studies/55-google-maps.md) | Geo-indexing, routing graphs, tile serving, ETA computation | ⬜ |
| 56 | [56-netflix-recommendation.md](./hld/case-studies/56-netflix-recommendation.md) | Collaborative filtering, real-time vs batch, A/B testing pipeline | ⬜ |
| 57 | [57-rag-llm-infrastructure.md](./hld/case-studies/57-rag-llm-infrastructure.md) | Vector DB sharding, embedding pipeline, multi-tenant LLM gateway | ⬜ |

---

### 🟠 LLD — LOW LEVEL DESIGN

---

#### 🧩 OOP Fundamentals

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 58 | [58-classes-and-objects.md](./lld/oop-fundamentals/58-classes-and-objects.md) | Classes, objects, constructors, destructors, memory layout, identity vs equality | ✅ |
| 59 | [59-encapsulation.md](./lld/oop-fundamentals/59-encapsulation.md) | Data hiding, getters/setters, access modifiers, name mangling, invariants | ✅ |
| 60 | [60-abstraction.md](./lld/oop-fundamentals/60-abstraction.md) | Hiding complexity, abstract classes, interfaces, leaky abstraction | ✅ |
| 61 | [61-inheritance.md](./lld/oop-fundamentals/61-inheritance.md) | Parent/child, method overriding, super(), MRO, diamond problem | ✅ |
| 62 | [62-polymorphism.md](./lld/oop-fundamentals/62-polymorphism.md) | Compile-time vs runtime, duck typing, operator overloading, generics | ✅ |
| 63 | [63-interfaces-vs-abstract-classes.md](./lld/oop-fundamentals/63-interfaces-vs-abstract-classes.md) | When to use which, ABC vs Protocol, interface segregation | ✅ |
| 64 | [64-composition-over-inheritance.md](./lld/oop-fundamentals/64-composition-over-inheritance.md) | Why composition scales better, has-a vs is-a, Strategy pattern, DI | ✅ |

---

#### 🏛️ SOLID Principles ✅ COMPLETED

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 65 | [65-single-responsibility.md](./lld/solid-principles/65-single-responsibility.md) | One reason to change, actor-based ownership, God Class anti-pattern | ✅ |
| 66 | [66-open-closed.md](./lld/solid-principles/66-open-closed.md) | Open for extension, closed for modification — Strategy pattern, plugin architecture | ✅ |
| 67 | [67-liskov-substitution.md](./lld/solid-principles/67-liskov-substitution.md) | Subclass substitutability, Square-Rectangle problem, 4 LSP rules | ✅ |
| 68 | [68-interface-segregation.md](./lld/solid-principles/68-interface-segregation.md) | Fat interfaces, role interfaces, Python Protocol, BFF pattern | ✅ |
| 69 | [69-dependency-inversion.md](./lld/solid-principles/69-dependency-inversion.md) | Depend on abstractions, DI containers, FastAPI Depends(), IoC | ✅ |

---

#### 🏗️ Design Patterns — Creational ✅ COMPLETED

| # | File | Problem it Solves | Status |
|---|------|-------------------|--------|
| 70 | [70-singleton.md](./lld/design-patterns/creational/70-singleton.md) | Only one instance — DB pool, config manager. Thread safety, DI alternative, testing | ✅ |
| 71 | [71-factory-method.md](./lld/design-patterns/creational/71-factory-method.md) | Create objects without specifying exact class — LLM provider factory, notification factory | ✅ |
| 72 | [72-abstract-factory.md](./lld/design-patterns/creational/72-abstract-factory.md) | Families of related objects — UI theme factory, cloud provider factory, DB factory | ✅ |
| 73 | [73-builder.md](./lld/design-patterns/creational/73-builder.md) | Build complex objects step by step — HTTP request builder, SQL query builder, prompt builder | ✅ |
| 74 | [74-prototype.md](./lld/design-patterns/creational/74-prototype.md) | Clone existing objects — game monster spawner, document templates, test data factories | ✅ |

---

#### 🏗️ Design Patterns — Structural

| # | File | Problem it Solves | Status |
|---|------|-------------------|--------|
| 75 | [75-adapter.md](./lld/design-patterns/structural/75-adapter.md) | Incompatible interfaces — wrapping Azure AI Search to a common interface | ⬜ |
| 76 | [76-decorator.md](./lld/design-patterns/structural/76-decorator.md) | Add behavior dynamically — logging/auth middleware in FastAPI | ⬜ |
| 77 | [77-facade.md](./lld/design-patterns/structural/77-facade.md) | Simplified interface to complexity — RAG service hiding retriever + LLM | ⬜ |
| 78 | [78-proxy.md](./lld/design-patterns/structural/78-proxy.md) | Control access — rate limiting proxy, caching proxy | ⬜ |
| 79 | [79-composite.md](./lld/design-patterns/structural/79-composite.md) | Treat individual and groups uniformly — file system tree | ⬜ |
| 80 | [80-bridge.md](./lld/design-patterns/structural/80-bridge.md) | Decouple abstraction from implementation — LLM interface from provider | ⬜ |
| 81 | [81-flyweight.md](./lld/design-patterns/structural/81-flyweight.md) | Share common state — token objects, glyph rendering | ⬜ |

---

#### 🏗️ Design Patterns — Behavioral

| # | File | Problem it Solves | Status |
|---|------|-------------------|--------|
| 82 | [82-observer.md](./lld/design-patterns/behavioral/82-observer.md) | Notify on state change — webhooks, Langfuse tracing hooks | ⬜ |
| 83 | [83-strategy.md](./lld/design-patterns/behavioral/83-strategy.md) | Swap algorithms at runtime — retrieval strategy in RAG | ⬜ |
| 84 | [84-chain-of-responsibility.md](./lld/design-patterns/behavioral/84-chain-of-responsibility.md) | Pass through handlers — middleware chain, guardrail pipeline | ⬜ |
| 85 | [85-command.md](./lld/design-patterns/behavioral/85-command.md) | Encapsulate request as object — task queues, undo/redo | ⬜ |
| 86 | [86-iterator.md](./lld/design-patterns/behavioral/86-iterator.md) | Traverse without exposing internals — streaming LLM tokens | ⬜ |
| 87 | [87-template-method.md](./lld/design-patterns/behavioral/87-template-method.md) | Skeleton algorithm, subclasses fill steps — document ingestion pipeline | ⬜ |
| 88 | [88-state.md](./lld/design-patterns/behavioral/88-state.md) | Behavior changes with internal state — chatbot conversation FSM | ⬜ |
| 89 | [89-mediator.md](./lld/design-patterns/behavioral/89-mediator.md) | Central coordinator between objects — chat room, air traffic control | ⬜ |
| 90 | [90-memento.md](./lld/design-patterns/behavioral/90-memento.md) | Capture and restore state — undo history, snapshot | ⬜ |

---

#### 📐 UML & Diagrams

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 91 | [91-class-diagrams.md](./lld/uml-and-diagrams/91-class-diagrams.md) | Classes, attributes, methods, relationships, cardinality | ⬜ |
| 92 | [92-sequence-diagrams.md](./lld/uml-and-diagrams/92-sequence-diagrams.md) | Object interaction over time, lifelines, activation bars | ⬜ |
| 93 | [93-use-case-diagrams.md](./lld/uml-and-diagrams/93-use-case-diagrams.md) | Actors, use cases, system boundary | ⬜ |
| 94 | [94-er-diagrams.md](./lld/uml-and-diagrams/94-er-diagrams.md) | Entities, attributes, relationships, crow's foot notation | ⬜ |

---

#### 🗄️ Database Schema Design

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 95 | [95-normalization.md](./lld/database-schema-design/95-normalization.md) | 1NF, 2NF, 3NF, BCNF, when to denormalize | ⬜ |
| 96 | [96-keys-and-constraints.md](./lld/database-schema-design/96-keys-and-constraints.md) | Primary, foreign, composite keys, unique and check constraints | ⬜ |
| 97 | [97-indexing-strategies.md](./lld/database-schema-design/97-indexing-strategies.md) | Single, composite, covering, partial indexes, index bloat | ⬜ |
| 98 | [98-relationships.md](./lld/database-schema-design/98-relationships.md) | One-to-one, one-to-many, many-to-many modeling | ⬜ |
| 99 | [99-junction-tables.md](./lld/database-schema-design/99-junction-tables.md) | Bridge tables, extra attributes on the join table | ⬜ |
| 100 | [100-soft-delete-and-timestamps.md](./lld/database-schema-design/100-soft-delete-and-timestamps.md) | is_deleted, deleted_at, created_at, updated_at patterns | ⬜ |
| 101 | [101-uuid-vs-autoincrement.md](./lld/database-schema-design/101-uuid-vs-autoincrement.md) | Tradeoffs, UUIDv4 vs UUIDv7, distributed ID generation | ⬜ |

---

#### 📡 API Contract Design

| # | File | What's Covered | Status |
|---|------|----------------|--------|
| 102 | [102-restful-principles.md](./lld/api-contract-design/102-restful-principles.md) | Nouns not verbs, HTTP methods, statelessness, HATEOAS | ⬜ |
| 103 | [103-http-status-codes.md](./lld/api-contract-design/103-http-status-codes.md) | 2xx, 3xx, 4xx, 5xx — when to use each exactly | ⬜ |
| 104 | [104-request-response-schema.md](./lld/api-contract-design/104-request-response-schema.md) | Consistent envelope, field naming, nullable vs optional | ⬜ |
| 105 | [105-versioning-strategies.md](./lld/api-contract-design/105-versioning-strategies.md) | URL versioning, header versioning, deprecation strategy | ⬜ |
| 106 | [106-pagination.md](./lld/api-contract-design/106-pagination.md) | Cursor-based vs offset, keyset pagination, total count tradeoff | ⬜ |
| 107 | [107-idempotency.md](./lld/api-contract-design/107-idempotency.md) | Idempotency keys, safe vs unsafe methods, retry safety | ⬜ |
| 108 | [108-error-response-standards.md](./lld/api-contract-design/108-error-response-standards.md) | RFC 7807 problem detail, error codes, machine-readable errors | ⬜ |

---

#### 🧪 LLD Practice Problems

##### Beginner

| # | File | Key Concepts | Status |
|---|------|-------------|--------|
| 109 | [109-parking-lot.md](./lld/practice-problems/beginner/109-parking-lot.md) | OOP hierarchy, strategy for pricing, singleton for lot manager | ⬜ |
| 110 | [110-library-management.md](./lld/practice-problems/beginner/110-library-management.md) | Book, Member, Loan entities, state machine for book status | ⬜ |
| 111 | [111-atm.md](./lld/practice-problems/beginner/111-atm.md) | State pattern, card/account/transaction classes | ⬜ |
| 112 | [112-vending-machine.md](./lld/practice-problems/beginner/112-vending-machine.md) | State machine, inventory management, payment handling | ⬜ |
| 113 | [113-chess-game.md](./lld/practice-problems/beginner/113-chess-game.md) | Board, Piece hierarchy, move validation, game state | ⬜ |

##### Intermediate

| # | File | Key Concepts | Status |
|---|------|-------------|--------|
| 114 | [114-hotel-booking.md](./lld/practice-problems/intermediate/114-hotel-booking.md) | Room types, booking lifecycle, pricing strategy, calendar availability | ⬜ |
| 115 | [115-elevator-system.md](./lld/practice-problems/intermediate/115-elevator-system.md) | Scheduler algorithm, state machine, direction and floor management | ⬜ |
| 116 | [116-food-delivery-app.md](./lld/practice-problems/intermediate/116-food-delivery-app.md) | Order lifecycle, restaurant/driver/customer entities, real-time tracking | ⬜ |
| 117 | [117-cab-booking.md](./lld/practice-problems/intermediate/117-cab-booking.md) | Matching algorithm, trip state, pricing, driver availability | ⬜ |
| 118 | [118-movie-ticket-booking.md](./lld/practice-problems/intermediate/118-movie-ticket-booking.md) | Seat locking, show/screen/theatre hierarchy, payment flow | ⬜ |

##### Advanced

| # | File | Key Concepts | Status |
|---|------|-------------|--------|
| 119 | [119-notification-service.md](./lld/practice-problems/advanced/119-notification-service.md) | Channel abstraction, observer pattern, retry, template engine | ⬜ |
| 120 | [120-lru-cache.md](./lld/practice-problems/advanced/120-lru-cache.md) | Doubly linked list + hashmap, O(1) get/put, thread safety | ⬜ |
| 121 | [121-llm-rag-pipeline.md](./lld/practice-problems/advanced/121-llm-rag-pipeline.md) | Retriever interface, reranker, LLM abstraction, guardrail chain | ⬜ |

---

## 🗺️ Study Path

```
START
  │
  ▼
hld/foundations/ (01 → 07)                        ✅ DONE
  │
  ▼
hld/components/ (08 → 15)                         ✅ DONE
  │
  ├─────────────────────────────────────────────────────────┐
  ▼                                                         ▼
hld/distributed-systems/ (16 → 29)          lld/oop-fundamentals/ (58 → 64)
✅ DONE                                           ✅ DONE
  │                                                         ▼
  ▼                                              lld/solid-principles/ (65 → 69)
hld/observability/ (30 → 37)                      ✅ DONE
✅ DONE
  │                                                         ▼
  ▼                                     lld/design-patterns/creational/ (70 → 74)
hld/security/ (38 → 41)                           ✅ DONE
✅ DONE
  │                                                         ▼
  ▼                                     lld/design-patterns/structural/ (75 → 81)
hld/case-studies/ (42 → 57)                                │
                                                            ▼
                                        lld/design-patterns/behavioral/ (82 → 90)
                                                            │
                                                            ▼
                                            lld/uml-and-diagrams/ (91 → 94)
                                                            │
                                                            ▼
                                            lld/database-schema-design/ (95 → 101)
                                                            │
                                                            ▼
                                            lld/api-contract-design/ (102 → 108)
                                                            │
                                                            ▼
                                            lld/practice-problems/ (109 → 121)
```

> Run HLD and LLD tracks in parallel — one topic from each per day.

---

## 🚀 How to Use This Repo

Each file follows this format:

```
## What is it?
## Why does it matter?
## How does it work?
## Trade-offs
## Real-world examples
## Interview Q&A
## Numbers to remember
```

**Self-testing technique:**
1. Read only the question, not the answer
2. Write or say your answer out loud
3. Check against the written answer
4. Focus on trade-offs — interviewers want *"it depends, and here's why"*, not textbook definitions

---

## 📦 Resources

| Resource | Type | Link |
|---|---|---|
| System Design Primer | GitHub | https://github.com/donnemartin/system-design-primer |
| System Design 101 | GitHub | https://github.com/ByteByteGoHq/system-design-101 |
| Awesome System Design Resources | GitHub | https://github.com/ashishps1/awesome-system-design-resources |
| Karan Pratap System Design | GitHub | https://github.com/karanpratapsingh/system-design |
| Awesome Low Level Design | GitHub | https://github.com/ashishps1/awesome-low-level-design |
| Python Patterns | GitHub | https://github.com/faif/python-patterns |
| Refactoring Guru | Website | https://refactoring.guru/design-patterns |
| ByteByteGo | YouTube | https://www.youtube.com/@ByteByteGo |
| ArjanCodes | YouTube | https://www.youtube.com/@ArjanCodes |
| Designing Data-Intensive Applications | Book | Martin Kleppmann |
| System Design Interview Vol 1 & 2 | Book | Alex Xu |
| Head First Design Patterns | Book | Freeman & Robson |

---

## 👤 Who This Is For

- Engineers preparing for system design interviews at senior/lead level
- Developers moving from individual services to distributed-systems thinking
- GenAI/backend engineers who want to formalize their architecture knowledge

---

## 📄 License

<p align="center">
  <a href="https://creativecommons.org/licenses/by-nc/4.0/">
    <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg" alt="License: CC BY-NC 4.0" />
  </a>
</p>

**© 2026 Aniket Waichal. All rights reserved.**

This work is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International License (CC BY-NC 4.0)**.

| | What it means |
|---|---|
| ✅ **Share** | Copy and redistribute in any medium or format |
| ✅ **Adapt** | Remix, transform, and build upon the material |
| ✅ **Personal use** | Study and use freely for your own learning |
| ❌ **No commercial use** | Cannot sell, include in paid courses, or monetize |
| ❌ **Must give credit** | Cannot publish or share without crediting Aniket Waichal |
| ❌ **No claiming ownership** | Cannot present this work as your own |

### Attribution Format

If you share or reference this work, please credit as:

> *"System Design Knowledge Base by **Aniket Waichal** — licensed under CC BY-NC 4.0"*

For commercial use or special permissions, contact the author directly.

Full license text → [creativecommons.org/licenses/by-nc/4.0](https://creativecommons.org/licenses/by-nc/4.0/legalcode)

---

```
Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)
Copyright (c) 2026 Aniket Waichal. All rights reserved.

You are free to share and adapt this material under the following terms:
  - Attribution: You MUST credit Aniket Waichal and link to this repository.
  - NonCommercial: You may NOT use this for commercial purposes.
  - No additional restrictions: You may not apply terms that restrict others
    from doing anything this license permits.

Full license: https://creativecommons.org/licenses/by-nc/4.0/legalcode
```