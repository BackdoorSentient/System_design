# 04_message_queues.md — Message Queues

---

**Q: Why use a message queue? What problems does it solve?**

A message queue decouples producers (services that generate events/data) from consumers (services that process them). Without a queue, a producer must call a consumer synchronously — if the consumer is down, the call fails.

**Problems solved**:

1. **Temporal decoupling**: Producer and consumer don't need to be running simultaneously. The message waits in the queue.
2. **Load leveling**: A sudden spike of 100,000 requests doesn't overwhelm the consumer. The queue buffers the burst and the consumer processes at its own pace.
3. **Reliability**: If the consumer crashes mid-processing, the message is re-delivered (at-least-once delivery).
4. **Fan-out**: One event can be consumed by multiple independent services (email service, analytics service, notification service) without the producer knowing about each of them.
5. **Backpressure**: Producers slow down if the queue fills up, preventing system-wide overload.

**Real-world examples**:
- Order placed → queue → inventory service, payment service, notification service (fan-out).
- Image uploaded → queue → thumbnail generation service (load leveling).
- User action → Kafka → analytics pipeline (event streaming).

---

**Q: Compare Kafka and RabbitMQ across all key dimensions.**

| Dimension | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed commit log (event streaming) | Message broker (queue) |
| Message retention | Retained on disk for configurable period (default 7 days) | Deleted after consumer ACKs |
| Replay | Yes — consumers can rewind and re-read | No — message is gone after processing |
| Consumer model | Pull-based (consumers control pace) | Push-based (broker pushes to consumers) |
| Ordering | Guaranteed within a partition | FIFO within a single queue |
| Throughput | Millions of msg/sec (sequential disk I/O, batching) | Hundreds of thousands msg/sec |
| Latency | Higher (batching adds latency, typically 5–15ms) | Lower (can be <1ms) |
| Consumer groups | Multiple independent groups reading same topic at different offsets | Competing consumers on a queue |
| Complex routing | Limited (topic-based, with keys) | Rich (exchanges: direct, topic, fanout, headers) |
| Persistence | Durable by design | Durable with persistent queues (performance cost) |
| Use cases | Event streaming, audit log, real-time analytics, event sourcing | Task queues, RPC, complex routing, low-latency messaging |

**Choose Kafka when**:
- You need to replay events (debugging, rebuilding derived state, new consumer bootstrapping).
- High throughput event streaming (clickstreams, logs, metrics).
- Multiple independent consumers need to read the same events.
- Event sourcing / CQRS architecture.

**Choose RabbitMQ when**:
- Task queues — worker pools processing jobs (resize image, send email).
- You need flexible routing (dead-letter queues, delayed messages, priority queues).
- Low latency per-message processing matters more than throughput.
- Complex routing topologies (multiple exchange types, binding patterns).

---

**Q: Explain pub/sub vs point-to-point messaging models.**

**Point-to-point (Queue model)**:
- One producer, one consumer. A message is delivered to exactly one consumer.
- Multiple consumers on the same queue compete for messages (competing consumers pattern).
- Each message is processed once.
- Used for: task distribution (job queues), load balancing work across workers.
- Example: RabbitMQ queue with 5 worker processes. Each job goes to exactly one worker.

**Pub/Sub (Publish-Subscribe)**:
- One producer (publisher), multiple consumers (subscribers).
- Each subscriber gets a copy of every message (or messages filtered by subscription criteria).
- Producer has no knowledge of subscribers.
- Used for: notifications, event broadcasting, fan-out.
- Example: Order placed event → inventory service, notification service, and analytics service all receive independent copies.

**Kafka's model**: Hybrid. A topic is published to (pub). Multiple consumer groups subscribe independently. Within a consumer group, partitions are divided among instances (competing consumers). So Kafka does pub/sub across groups and point-to-point within a group.

---

**Q: What is the difference between at-most-once, at-least-once, and exactly-once delivery?**

**At-most-once**: Messages may be lost, but never duplicated. Producer fires and forgets; if the broker crashes before persisting, the message is gone. Simple and fast. Acceptable for: metrics, logs where occasional loss is tolerable.

**At-least-once**: Messages are never lost, but may be delivered multiple times. The consumer must ACK the message; if it crashes before ACKing, the broker re-delivers. Duplicates are possible. Most production systems use this. Consumer must be **idempotent** — processing the same message twice has the same effect as once.

**Exactly-once**: Delivered exactly once, no loss and no duplicates. Hardest to achieve. Requires coordination between producer, broker, and consumer.

**Kafka's exactly-once semantics (EOS)**:
- **Idempotent producer**: Each message has a producer ID + sequence number. Broker deduplicates retries from the same producer (at the partition level).
- **Transactional producer**: Multiple writes to different partitions are atomic — either all succeed or none do. Used with `beginTransaction()` / `commitTransaction()`.
- **Read-process-write**: In Kafka Streams, reading input, processing, and writing output + committing offsets happen atomically in one transaction.
- Performance cost: ~3–5x latency increase vs non-transactional. Use only when correctness is critical (financial data, exactly-once state updates).

**In practice**: At-least-once + idempotent consumers is the dominant production pattern. True exactly-once is used selectively.

---

**Q: How do Kafka partitions work? How do they enable parallelism and ordering guarantees?**

A Kafka **topic** is split into **partitions** — ordered, immutable sequences of messages stored on disk.

**Parallelism**: Each partition can be consumed by at most one consumer in a consumer group simultaneously. So if a topic has 12 partitions, a consumer group can scale to at most 12 consumers (instances), each processing a partition in parallel.

**Ordering**: Messages are strictly ordered within a partition. Order is not guaranteed across partitions. If you need all messages for a specific entity (user, order) to be processed in order, route them to the same partition using a consistent **partition key** (e.g., `user_id`). All events for user 123 go to partition `hash(123) % num_partitions`.

**Replication**: Each partition has a configurable replication factor (typically 3). One replica is the leader (handles reads and writes); others are followers. If the leader fails, a follower is elected leader (ISR — In-Sync Replica set).

**Offset**: A sequential ID for each message within a partition. Consumers track their position via committed offsets. Reprocessing is as simple as resetting the offset to an earlier position.

**Sizing guidelines**:
- Partition count: Start with `max(desired_throughput / throughput_per_consumer, desired_parallelism)`.
- Typical production: 10–100 partitions per topic.
- More partitions = more parallelism, but more overhead (each partition is an open file, leader election takes longer).
- Rule of thumb from LinkedIn: ~10 partitions per broker at moderate load.

---

**Q: What are dead letter queues (DLQ) and why are they essential?**

A Dead Letter Queue (DLQ) is a special queue where messages are routed when they fail to be processed successfully after a configurable number of retries.

**Why they matter**: Without a DLQ, a poison pill message (malformed data, unexpected format, logic error) causes a consumer to crash or infinite-retry, blocking all subsequent messages in the queue. The DLQ quarantines the bad message and lets processing continue.

**Typical flow**:
1. Consumer receives message and throws an exception.
2. Broker re-delivers after a backoff (e.g., 30 seconds).
3. After 3–5 retries, the message is moved to the DLQ.
4. The consumer continues processing subsequent messages.
5. An operator inspects the DLQ, fixes the bug, and re-processes the messages.

**In RabbitMQ**: Configure `x-dead-letter-exchange` on a queue. Failed messages are routed to the specified exchange/queue.

**In AWS SQS**: Set `maxReceiveCount` on the queue's redrive policy. After N failed receives, the message is moved to a designated DLQ.

**In Kafka**: No native DLQ concept. Common pattern: in the consumer's error handler, publish the failed message to a `topic.DLQ` topic manually.

---

**Q: What is consumer group rebalancing in Kafka and what are its performance implications?**

When a consumer joins or leaves a consumer group (due to scale-up, instance failure, or deployment), Kafka must **rebalance** — redistribute partition assignments among the active consumers.

**Rebalance process (eager/stop-the-world)**:
1. Group coordinator (a Kafka broker) detects membership change.
2. Revokes all partition assignments from all consumers (they stop processing).
3. Assigns partitions to current consumers.
4. Consumers resume processing.

**Impact**: During rebalance (typically 10–30 seconds), no messages are processed. For high-throughput systems, this is a significant availability gap.

**Mitigations**:
- **Static group membership** (`group.instance.id`): Consumers are identified by a stable ID. A restart/redeploy doesn't trigger a rebalance (the consumer is considered temporarily offline, not left the group). If it rejoins within `session.timeout.ms`, it gets its old partitions back.
- **Incremental cooperative rebalancing** (Kafka 2.4+): Only revokes partitions that need to move. Consumers not involved continue processing. Reduces downtime significantly.
- **Proper `session.timeout.ms`**: Default 10 seconds. Long enough that temporary GC pauses don't trigger false rebalances; short enough that dead consumers are detected quickly.

---

**Q: What are the key Kafka producer configuration parameters and their trade-offs?**

**`acks`**:
- `acks=0`: Producer doesn't wait for any acknowledgment. Maximum throughput, zero durability. Messages can be lost.
- `acks=1`: Producer waits for the leader to acknowledge. Message persisted to leader. Leader failure before replication = data loss.
- `acks=all` (or `-1`): Producer waits for all in-sync replicas to acknowledge. Maximum durability. Higher latency (~2–10ms extra for replication).

**`retries` and `delivery.timeout.ms`**:
- Enable retries for transient failures. With idempotent producer, retries are safe.
- `delivery.timeout.ms` = total time budget for a message to be delivered (default: 2 minutes).

**`batch.size` and `linger.ms`**:
- Producer batches messages destined for the same partition.
- `batch.size`: Maximum bytes per batch (default: 16KB). Larger batches = better throughput, more latency.
- `linger.ms`: How long to wait before sending a partial batch (default: 0ms — send immediately). Increase to 5–50ms to improve batching efficiency at the cost of latency.

**`compression.type`**: `snappy` or `lz4` for throughput. `gzip` for highest compression ratio. Compression reduces network bandwidth and disk usage (3–10x for text/JSON).

**`max.in.flight.requests.per.connection`**: With retries enabled, setting this >1 can cause message reordering (retry of message 1 after message 2 succeeds). Set to 1 if ordering is critical, or use idempotent producer (allows up to 5 in-flight without reordering).
