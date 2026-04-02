# 20. Distributed Transactions — 2PC & Sagas

## What is a Distributed Transaction?

A distributed transaction ensures that a sequence of operations across **multiple services or databases** either all succeed or all fail — maintaining ACID properties across boundaries.

**Example:** Transfer $100 from Account A (Bank DB) to Account B (Bank DB):
- Debit A: -$100
- Credit B: +$100
Both must succeed or neither should (no partial state).

When A and B are on different databases or microservices, you can't use a local transaction. You need a distributed coordination mechanism.

---

## Q1: How does Two-Phase Commit (2PC) work?

2PC is a blocking protocol that coordinates all participants to either commit or rollback a transaction atomically.

### Roles
- **Coordinator:** Orchestrates the protocol (usually the transaction manager or the initiating service)
- **Participants:** Services/databases involved in the transaction

### Phase 1: Prepare (Voting Phase)

1. Coordinator sends `PREPARE` to all participants
2. Each participant:
   - Executes the transaction locally (but doesn't commit)
   - Writes to its undo log (in case rollback is needed)
   - Responds `YES` (ready to commit) or `NO` (abort)
3. Participants wait for coordinator's decision — they are **in doubt**

### Phase 2: Commit / Abort

**If ALL voted YES:**
1. Coordinator logs `COMMIT` to its durable log
2. Sends `COMMIT` to all participants
3. Participants commit, release locks, acknowledge
4. Coordinator marks transaction done

**If ANY voted NO:**
1. Coordinator sends `ABORT` to all
2. Participants roll back using undo log

```
Coordinator        Participant A      Participant B
     |──PREPARE──────►|                    |
     |──PREPARE───────────────────────────►|
     |◄──YES──────────|                    |
     |◄──YES──────────────────────────────|
     |──COMMIT────────►|                   |
     |──COMMIT────────────────────────────►|
     |◄──ACK──────────|                    |
     |◄──ACK──────────────────────────────|
```

### 2PC Problems

**1. Blocking protocol — coordinator is a SPOF:**
If the coordinator crashes after Phase 1 but before Phase 2, participants are stuck "in doubt" — they've voted YES and can't release locks until the coordinator recovers.

**2. Blocking participants hold locks:**
During the in-doubt window, participants hold row-level locks. Under high concurrency, this causes lock contention and timeouts.

**3. Doesn't tolerate network partitions:**
If coordinator can't reach participants during Phase 2, transaction is stuck until network recovers.

**Trade-offs:**

| Pro | Con |
|-----|-----|
| Strong ACID guarantees | Coordinator SPOF |
| Atomic commit across multiple DBs | Blocking on coordinator failure |
| Widely supported | Locks held during uncertain period |
| | Poor performance at scale |

**Used by:** MySQL XA transactions, PostgreSQL 2PC, many enterprise databases.

---

## Q2: What is Three-Phase Commit (3PC)?

3PC adds an intermediate phase to reduce the blocking problem of 2PC.

**Three phases:** Prepare → Pre-commit → Commit

In the pre-commit phase, participants know everyone voted YES. If the coordinator fails now, any participant can safely take over (they all know the outcome). This reduces (but doesn't eliminate) blocking.

**In practice:** 3PC is rarely used — it's more complex, still doesn't handle network partitions, and the performance improvement isn't worth it. Sagas are preferred for microservices.

---

## Q3: What are Sagas?

A **saga** is a sequence of local transactions, each publishing an event or message that triggers the next. If one step fails, compensating transactions are executed to undo previous steps.

**Key insight:** Instead of one big atomic transaction, break it into small local transactions with compensation logic.

### Example: E-Commerce Order (5 steps)

```
1. Create Order          → compensate: Cancel Order
2. Reserve Inventory     → compensate: Release Inventory
3. Charge Payment        → compensate: Refund Payment
4. Schedule Shipping     → compensate: Cancel Shipping
5. Send Confirmation     → (no compensation needed)
```

If step 4 (Schedule Shipping) fails:
- Execute compensations in reverse: Refund Payment → Release Inventory → Cancel Order

### Two Saga Patterns

**Choreography (Event-driven):**
Each service publishes events after completing its step. Other services listen and react.

```
Order Service ──order.created──► Inventory Service
                                        │
                              ──inventory.reserved──► Payment Service
                                                             │
                                               ──payment.charged──► Shipping Service
```

**Pros:** Loose coupling, no central coordinator
**Cons:** Hard to visualize the overall flow, debugging distributed failures is painful

---

**Orchestration (Central coordinator):**
A saga orchestrator explicitly calls each service and handles failures.

```
         ┌─────────────────────────────────┐
         │         Saga Orchestrator        │
         │                                 │
         │  1. call Order Service          │
         │  2. call Inventory Service      │
         │  3. call Payment Service        │
         │  4. call Shipping Service       │
         │  on failure: run compensations  │
         └─────────────────────────────────┘
```

**Pros:** Easy to visualize, centralized error handling, can implement complex logic
**Cons:** Orchestrator is a single point of responsibility (not failure — it can be replicated)

---

## Q4: 2PC vs Sagas — When to use which?

| | 2PC | Sagas |
|---|---|---|
| Consistency | **Strong (ACID)** | Eventual (BASE) |
| Isolation | Full (locks) | None (dirty reads between steps) |
| Failure handling | Auto-rollback | Compensating transactions (manual) |
| Performance | Low (blocking, lock contention) | High (no distributed locks) |
| Cross-DB support | Yes (XA) | Requires messaging |
| Use case | Financial core systems, same-org multi-DB | Microservices, long-running workflows |

---

## Q5: What is the Outbox Pattern?

Sagas require reliable event publishing — if a service completes a local transaction but crashes before publishing the event, the saga halts.

**Outbox Pattern:**
1. In the same local transaction that updates the DB, also write the event to an `outbox` table
2. A separate poller reads the outbox and publishes to the message queue
3. On publish success, delete from outbox (or mark as published)

```sql
BEGIN TRANSACTION;
UPDATE orders SET status='confirmed' WHERE id=123;
INSERT INTO outbox (event_type, payload) VALUES ('order.confirmed', '{"id":123}');
COMMIT;
-- Separate process reads outbox → publishes to Kafka → deletes row
```

This ensures **at-least-once delivery** without distributed transactions. The event is published exactly when and only when the DB transaction commits.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| 2PC typical latency overhead | 2–4 extra network round trips |
| 2PC lock hold time (worst case) | Until coordinator recovers (minutes) |
| Saga compensation time | Depends on number of steps (seconds to minutes) |
| Outbox poll interval (typical) | 100ms – 1s |

---

## Real-World Examples

| System | Approach |
|--------|----------|
| Bank core systems | 2PC with XA transactions |
| Amazon order workflow | Saga (choreography) |
| Uber trip lifecycle | Saga (orchestration with Cadence/Temporal) |
| Netflix checkout | Saga with compensation |
| Temporal / Cadence | Platform for implementing saga orchestration |

---

## Interview Q&A

**Q: Why don't microservices use 2PC?**
A: 2PC requires all participating services to hold locks during the transaction. In microservices, this means one slow service (or network timeout) can block all others, causing cascading failures. It also requires a coordinator that becomes a SPOF. Microservices use sagas with compensation instead — accepting eventual consistency in exchange for availability and performance.

**Q: How do you handle a failed compensation in a saga?**
A: Compensations should be idempotent and retried indefinitely (with exponential backoff). If a compensation truly cannot succeed (e.g., a payment was already refunded manually), you need a human-in-the-loop escalation path — write to a dead-letter queue and alert the ops team. Design compensations to be retryable from the start.

**Q: What isolation issues exist in sagas?**
A: Sagas don't provide isolation between concurrent sagas. Two sagas might read the same data at the same time and make conflicting changes. This is a "lost update" or "dirty read" at the saga level. Mitigations: semantic locking (mark a record as "in-saga"), pessimistic ordering (process one saga at a time for a given resource), or accept eventual consistency and use reconciliation.