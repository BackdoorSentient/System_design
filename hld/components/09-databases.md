# 03_databases.md — Databases

---

**Q: What are the fundamental differences between SQL and NoSQL databases?**

| Dimension | SQL (Relational) | NoSQL |
|---|---|---|
| Schema | Fixed, enforced schema | Flexible / schema-less |
| Data model | Tables with rows and foreign keys | Document, key-value, wide-column, graph |
| Query language | Standardized SQL | Varies by DB |
| Joins | First-class | Avoided; denormalization preferred |
| Transactions | ACID across multiple tables | Usually single-document/row; some support multi-doc |
| Scaling | Vertical primary; horizontal is hard | Horizontal-first design |
| Consistency | Strong (ACID) | Eventual (BASE) typically |
| Examples | PostgreSQL, MySQL, Oracle, SQLite | MongoDB, DynamoDB, Cassandra, Redis, Neo4j |

The "NoSQL vs SQL" framing is outdated. Modern PostgreSQL supports JSON documents, full-text search, graph queries (via recursive CTEs), and time-series extensions. The real question is: which data model and consistency model fits your use case?

---

**Q: What is ACID? Explain each property with a real-world example.**

ACID guarantees that database transactions are processed reliably.

**Atomicity**: A transaction is all-or-nothing. If a bank transfer deducts $100 from account A and credits $100 to account B, either both operations succeed or neither does. A crash mid-transaction leaves no partial state.

**Consistency**: A transaction takes the DB from one valid state to another. All defined constraints, rules, and cascades are respected. If a foreign key constraint says `orders.customer_id` must exist in `customers`, a transaction inserting an order with a non-existent customer is rejected.

**Isolation**: Concurrent transactions behave as if they were serial. Transaction T1 reading a row while T2 is updating it sees either the pre-update or post-update value, never a partial write. SQL defines four isolation levels:
- **Read Uncommitted**: Sees uncommitted changes. Dirty reads possible.
- **Read Committed**: Only sees committed data. Non-repeatable reads possible (row changes between two reads in same transaction).
- **Repeatable Read**: Same row always returns same value within a transaction. Phantom reads possible (new rows can appear).
- **Serializable**: Full isolation. Equivalent to serial execution. Most expensive.

Most databases default to Read Committed. PostgreSQL's default is Read Committed; MySQL InnoDB defaults to Repeatable Read.

**Durability**: Once a transaction is committed, it persists even if the system crashes immediately after. Achieved via write-ahead logging (WAL) — the log entry is fsynced to disk before the transaction is acknowledged.

---

**Q: What is BASE and how does it differ from ACID?**

BASE stands for **Basically Available, Soft state, Eventually consistent**. It describes the consistency model of most distributed NoSQL databases.

- **Basically Available**: The system remains available (returns a response) even if some nodes are down. Reads may return stale data rather than refusing.
- **Soft state**: The system state may change over time even without new input (as replication propagates changes).
- **Eventually consistent**: Given no new updates, all replicas will eventually converge to the same value.

**ACID vs BASE trade-off**: ACID prioritizes correctness over availability. In a distributed system, enforcing ACID-level consistency across nodes requires coordination (2-phase commit, Paxos) which increases latency and reduces availability. BASE accepts staleness in exchange for low latency and high availability.

**Real-world**: Amazon DynamoDB is BASE by default. If you read immediately after a write, you might get the old value from a different replica. You can opt into strongly consistent reads (at 2x the cost). Cassandra lets you tune consistency level per query (`QUORUM`, `ONE`, `ALL`).

---

**Q: When do you choose SQL vs NoSQL? Give concrete use case examples.**

**Use SQL when**:
- Data is relational with complex join requirements (e.g., e-commerce: orders, line items, customers, products).
- You need complex ad-hoc queries or reporting.
- Strong transactional consistency is required (financial systems, inventory management).
- Schema is stable and well-understood.
- **Examples**: Banking (PostgreSQL), ERP systems (Oracle), analytics (Redshift/BigQuery which is SQL but columnar).

**Use NoSQL when**:
- Data is hierarchical or document-oriented and naturally fits a single document (e.g., user profile with all their preferences as one JSON blob).
- You need horizontal write scaling across many nodes (Cassandra for IoT time-series data).
- Schema is dynamic or varies per record (product catalog where different product types have different attributes).
- You need very low latency key-value lookups (Redis, DynamoDB).
- Graph relationships are first-class (Neo4j for social graphs, fraud detection).

**Specific NoSQL sub-types**:
- **Document** (MongoDB, DynamoDB): Content management, catalogs, user profiles.
- **Key-value** (Redis, DynamoDB): Session stores, rate limiting, caching.
- **Wide-column** (Cassandra, HBase): Time-series data, write-heavy analytics, IoT, logs. Optimized for reads by a known partition key.
- **Graph** (Neo4j, Amazon Neptune): Social networks, recommendation engines, fraud detection.

**Hybrid**: Many modern systems use both. SQL for transactional data, Redis for caching, Elasticsearch for full-text search, Cassandra for time-series events.

---

**Q: Explain database indexing deeply. What are B-tree and hash indexes?**

An index is a separate data structure maintained alongside a table to enable faster lookups without scanning every row.

**B-tree index** (the default in PostgreSQL, MySQL, Oracle):
- A balanced tree where each node contains sorted keys and pointers.
- Supports: equality (`=`), range (`>`, `<`, `BETWEEN`), prefix matching, sorting (`ORDER BY`).
- O(log N) lookup. Height of tree for 1 billion rows: ~30 levels.
- Leaf nodes are linked (B+ tree), enabling efficient range scans.
- Write overhead: Every INSERT/UPDATE/DELETE must update the index. For a table with 5 indexes, each write updates 6 data structures.

**Hash index**:
- Hash the key to find the bucket containing the value pointer.
- O(1) average lookup.
- Only supports equality (`=`). Cannot do range queries or sorting.
- Fits entirely in memory ideally. Used by PostgreSQL for in-memory hash indexes and by most key-value stores.

**Composite indexes**: Index on (col_A, col_B). Efficient for queries filtering on col_A alone or both (col_A, col_B). NOT efficient for col_B alone (left-prefix rule).

**Covering index**: An index that contains all columns needed to satisfy a query. No table lookup needed. The query is answered entirely from the index.

**Index selectivity**: The fraction of distinct values in a column. A boolean column (true/false) has low selectivity — an index on it is almost useless. `user_id` (unique) has high selectivity — ideal for indexing.

**Partial indexes**: Index only rows matching a condition: `CREATE INDEX ON orders (created_at) WHERE status = 'pending'`. Smaller, faster, more targeted.

**The index bloat trade-off**:
- More indexes → faster reads.
- More indexes → slower writes, more disk space, more memory for the index.
- Rule of thumb: Index columns that appear in `WHERE`, `JOIN ON`, and `ORDER BY` clauses with high selectivity. Avoid over-indexing write-heavy tables.

---

**Q: What is database normalization and when should you denormalize?**

**Normalization** eliminates data redundancy by organizing data into related tables linked by foreign keys.

- **1NF**: No repeating groups; each column has atomic values.
- **2NF**: No partial dependencies on composite primary key.
- **3NF**: No transitive dependencies (non-key column depends only on the primary key, not on another non-key column).

**Benefits of normalization**: No update anomalies (change a customer's name in one place, not 1,000 order rows). Smaller rows = more rows per page = better cache efficiency.

**Denormalization** intentionally introduces redundancy for read performance.

- **When to denormalize**: High-read, low-write workloads. When joins are too expensive (e.g., joining 5 tables for every API request). Data warehousing (star schema — fact tables surrounded by dimension tables).
- **Example**: Instead of joining `orders JOIN customers JOIN addresses` on every order read, store `customer_name` and `shipping_address` directly in the orders row.
- **Costs**: Update anomalies — you must update the denormalized copy wherever it exists. More storage. More complex write logic.

**NoSQL effectively forces denormalization** — since joins aren't supported, you model data as documents that embed related data.

---

**Q: What is the N+1 query problem and how do you fix it?**

**The problem**: When loading a list of N items, the ORM issues 1 query to load the list and then N additional queries to load related data for each item.

Example:
```sql
-- 1 query to load 100 orders
SELECT * FROM orders LIMIT 100;
-- Then 100 queries: one per order to load the customer
SELECT * FROM customers WHERE id = ?;  -- repeated 100 times
```

This is 101 queries instead of 1 or 2.

**Fixes**:
1. **Eager loading / JOIN**: Load orders and customers in a single query with a JOIN.
2. **Batch loading (DataLoader pattern)**: Collect all `customer_id` values, then issue one `SELECT * FROM customers WHERE id IN (1, 2, 3, ...)`. Used by Facebook's DataLoader in GraphQL.
3. **ORM eager loading**: Most ORMs support `.includes()` or `.prefetch_related()` to prevent N+1 automatically.

---

**Q: What is the CAP theorem and how does it apply to database choices?**

CAP theorem states that a distributed system can guarantee at most 2 of these 3 properties simultaneously:

- **Consistency (C)**: Every read returns the most recent write.
- **Availability (A)**: Every request receives a response (may not be the most recent data).
- **Partition tolerance (P)**: System continues operating despite network partitions.

Since network partitions are a reality in distributed systems, you must choose between **CP** or **AP**.

**CP systems** (prioritize consistency over availability during a partition):
- During a partition, some nodes refuse to respond rather than return stale data.
- Examples: HBase, Zookeeper, Etcd, traditional RDBMS with synchronous replication.

**AP systems** (prioritize availability over consistency during a partition):
- During a partition, nodes return possibly stale data rather than refusing.
- Examples: Cassandra, DynamoDB (default), CouchDB.

**Nuance**: CAP is binary and oversimplified. PACELC (extends CAP) also considers the latency/consistency trade-off even when there is no partition. In practice, databases offer tunable consistency (Cassandra's consistency levels, DynamoDB's strongly consistent reads).

---

**Q: How does database replication work? What are synchronous vs asynchronous replication trade-offs?**

**Replication** propagates writes from the primary (leader) to one or more replicas (followers).

**Asynchronous replication** (most common):
- Primary commits the write and acknowledges the client immediately. Replica receives and applies the change shortly after.
- Pros: Low write latency (no waiting for replicas). Replicas can be geographically distant.
- Cons: If the primary fails before the replica catches up, committed data is lost (replication lag creates potential data loss window). Replica reads may be stale.
- Used by: MySQL default, PostgreSQL `async` replication.

**Synchronous replication**:
- Primary waits for at least one replica to confirm the write before acknowledging the client.
- Pros: Zero data loss — at least one replica always has the latest data.
- Cons: Write latency = primary write + network RTT to replica + replica write. If the replica is slow or down, writes stall.
- PostgreSQL: `synchronous_commit = on`. AWS RDS Multi-AZ uses synchronous replication between primary and standby.

**Semi-synchronous**: Wait for one replica, not all. Good middle ground. MySQL supports this.

**Replication lag** monitoring: Check `seconds_behind_master` in MySQL or `pg_stat_replication.replay_lag` in PostgreSQL. Alert if lag > 30 seconds for critical services.

**Read replicas for scaling**: Route read traffic to replicas. Amazon RDS supports up to 15 read replicas. Be aware that reads may be slightly stale (eventual consistency).
