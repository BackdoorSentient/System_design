# 21. Event-Driven Architecture — Event Sourcing & CQRS

## What is Event-Driven Architecture (EDA)?

In EDA, services communicate by producing and consuming **events** (immutable facts about something that happened) rather than making direct synchronous calls.

**Event:** "OrderPlaced { orderId: 123, userId: 456, items: [...] }"
**vs Command:** "PlaceOrder(...)" — a request to do something

Events are facts about the past. They decouple producers from consumers.

---

## Q1: What is Event Sourcing?

Instead of storing the **current state** of an entity, store the **full sequence of events** that led to that state. The current state is derived by replaying events.

### Traditional State Storage:

```
orders table:
| id  | status    | total | updated_at |
|-----|-----------|-------|------------|
| 123 | shipped   | $99   | 2024-01-05 |
```

### Event Sourcing Storage:

```
order_events table:
| id  | order_id | event_type       | payload                      | timestamp  |
|-----|----------|------------------|------------------------------|------------|
| 1   | 123      | OrderPlaced      | {items:[...], total:99}      | 2024-01-01 |
| 2   | 123      | PaymentConfirmed | {txn_id: abc}                | 2024-01-02 |
| 3   | 123      | OrderShipped     | {tracking: XYZ}              | 2024-01-05 |
```

Current state = replay all events for order 123.

### Benefits

1. **Complete audit trail** — every state change is recorded, immutable
2. **Time-travel** — reconstruct state at any point in time
3. **Event replay** — rebuild projections (read models) from scratch
4. **Debugging** — understand exactly how a bug occurred
5. **Integration** — new services can replay history to bootstrap

### Drawbacks

1. **Eventual consistency** — projections lag behind the event stream
2. **Eventual large event log** — must use snapshots to avoid replaying all events
3. **Schema evolution** — old events must remain deserializable as schemas change
4. **Conceptual complexity** — harder to query "what is the current state?"

### Snapshots

To avoid replaying 10,000 events for an old order:
- Periodically snapshot the current state
- On read: load latest snapshot + replay only events after the snapshot

```
Snapshot at event #500: { status: "paid", total: $99 }
Replay events #501–#503 to get current state
```

---

## Q2: What is CQRS (Command Query Responsibility Segregation)?

CQRS separates the **write model** (commands that mutate state) from the **read model** (queries that return data).

```
           Commands (writes)               Queries (reads)
Client ──► [Command Handler] ──► Write DB  ──► [Query Handler] ──► Read DB
                                    │                                  ▲
                                    └──── Event Bus ──────────────────┘
                                          (updates read model)
```

### Why CQRS?

- Write model is normalized (optimized for integrity)
- Read model is denormalized (optimized for query speed)
- Read and write workloads can scale independently
- Read replicas can use different DB technologies (e.g., Elasticsearch for full-text search)

### Example: Social Feed

**Write model (Postgres):**
```
posts: { post_id, user_id, content, created_at }
follows: { follower_id, following_id }
```

**Read model (Redis / Cassandra):**
```
user_feed:{user_id} → [post_id_1, post_id_2, ...]  (pre-computed fan-out)
```

When a new post is written:
1. Write to `posts` table
2. Publish `PostCreated` event
3. Fan-out service consumes event → updates each follower's feed in Redis

Reads are instant (pre-computed). Writes trigger async updates.

---

## Q3: How do Event Sourcing and CQRS work together?

They're complementary but independent:

- **Event Sourcing alone:** Store events as source of truth, derive state by replay. No mandatory CQRS.
- **CQRS alone:** Separate read/write models. No mandatory event sourcing.
- **Together:** Events are the write model. Multiple read projections consume the event stream.

```
Command → Aggregate → Emit Events → Event Store
                                         │
                                    ─────┼────────
                                    │    │       │
                              Projection1 Proj2  Proj3
                               (MySQL)  (Redis) (Elastic)
```

Each projection is a different read model optimized for different queries.

---

## Q4: What is the Outbox Pattern in EDA?

(See also: Distributed Transactions topic)

The fundamental problem: How do you atomically update your database AND publish an event?

**Wrong approach:**
```
1. UPDATE db (succeeds)
2. Publish to Kafka (fails → event never sent)
```

**Outbox Pattern:**
```
BEGIN TRANSACTION;
  UPDATE orders SET status = 'confirmed';
  INSERT INTO outbox (event_type, payload) VALUES ('OrderConfirmed', '...');
COMMIT;
-- Background worker reads outbox → publishes to Kafka → marks as sent
```

The outbox guarantees at-least-once delivery. Make consumers idempotent to handle duplicates.

---

## Q5: What are the trade-offs of EDA?

| Benefit | Cost |
|---------|------|
| Loose coupling between services | Eventual consistency |
| Independent scaling of consumers | Harder to debug distributed flows |
| Natural audit log (with event sourcing) | Schema evolution complexity |
| Replay to bootstrap new services | Operational complexity (Kafka, etc.) |
| High throughput (async) | Out-of-order event handling needed |

---

## Q6: What is event ordering and how do you handle it?

Events from different producers may arrive out of order.

**Strategies:**

1. **Partition by entity ID** (Kafka): All events for order #123 go to the same partition → guaranteed ordering per entity.

2. **Sequence numbers / version vectors:** Each event has a sequence number. Consumers check that sequence N-1 is processed before N.

3. **Idempotent consumers:** Design consumers to handle duplicate or out-of-order events gracefully. "Apply this event only if current version == expected version."

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Typical event processing lag | 10–500ms (Kafka consumer) |
| Event store growth rate (active system) | Can reach billions of events/year |
| Snapshot frequency (typical) | Every 100–1000 events per aggregate |
| Kafka retention default | 7 days |

---

## Real-World Examples

| System | Pattern |
|--------|---------|
| Axon Framework | Event sourcing + CQRS (Java) |
| Netflix | EDA with Kafka for decoupling services |
| LinkedIn | Kafka as event backbone |
| Amazon | Event sourcing in order management |
| Shopify | Outbox pattern for reliable event publishing |
| Martin Fowler's LMAX | EDA + event sourcing for low-latency trading |

---

## Interview Q&A

**Q: When would you NOT use event sourcing?**
A: When you need simple CRUD with no audit requirements, when the team isn't experienced with it (high learning curve), or when the domain doesn't benefit from history (e.g., a user preferences table). Event sourcing adds significant complexity — use it only when time-travel, audit trail, or event-driven integration is a core requirement.

**Q: How does CQRS help with scaling a social media feed?**
A: On Twitter/Instagram, reads (millions of users loading feed) vastly outnumber writes (posting). With CQRS, the write model stores posts and follow relationships, and fan-out workers pre-compute each user's feed into a read store (Redis). Feed reads are O(1). Without CQRS, generating a feed at read time requires joining posts and follows — slow at scale.

**Q: How do you evolve event schemas without breaking old consumers?**
A: Use schema versioning (v1, v2 events), add fields as optional and never remove/rename fields. Use a schema registry (Confluent Schema Registry for Avro/Protobuf) to enforce compatibility. For breaking changes, run old and new consumers in parallel during a transition period, then retire old format.