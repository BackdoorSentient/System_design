# 25. Bloom Filters & Probabilistic Data Structures

## What is a Probabilistic Data Structure?

A probabilistic data structure trades **perfect accuracy** for dramatic savings in memory or time. It can answer certain queries approximately — with a controlled probability of error.

**Core trade-off:** "I'll accept a small chance of a wrong answer to use 1000× less memory."

---

## Q1: What is a Bloom Filter?

A Bloom filter is a space-efficient data structure that answers the question: **"Is this element in the set?"**

- **Definite NO** — if Bloom filter says "not in set," the element is definitely absent
- **Probable YES** — if Bloom filter says "in set," the element is *probably* there (false positives possible)
- **No false negatives** — it never says "not in set" for an element that is actually in the set

### Structure

A Bloom filter is a **bit array** of M bits (initially all 0) and K hash functions.

**Inserting an element:**
1. Hash the element with K hash functions → get K positions in the bit array
2. Set all K bits to 1

**Querying an element:**
1. Hash with same K functions → get K positions
2. If ALL K bits are 1 → "probably in set"
3. If ANY bit is 0 → "definitely not in set"

```
M = 10 bits, K = 3 hash functions

Insert "hello":
  h1("hello") = 2
  h2("hello") = 5
  h3("hello") = 8
  Bit array: 0 0 1 0 0 1 0 0 1 0
                 ^     ^     ^

Query "world":
  h1("world") = 1
  h2("world") = 5  ← bit is 1 (set by "hello")
  h3("world") = 9
  Bit 1 = 0 → "definitely NOT in set" ✓

Query "hello":
  All 3 bits (2, 5, 8) are 1 → "probably in set" ✓

Query "foobar":
  h1("foobar") = 2  ← set by "hello"
  h2("foobar") = 5  ← set by "hello"
  h3("foobar") = 8  ← set by "hello"
  All 3 are 1 → "probably in set" ← FALSE POSITIVE!
```

### False Positive Rate

As more elements are inserted, more bits are set → higher false positive rate.

**Optimal parameters:**
```
False positive probability p = (1 - e^(-kn/m))^k

Where:
  n = number of elements to insert
  m = number of bits
  k = number of hash functions

Optimal k = (m/n) * ln(2) ≈ 0.693 * (m/n)
```

**Rule of thumb:** 10 bits per element → ~1% false positive rate.

---

## Q2: Where is Bloom filter used in real systems?

| System | Use Case |
|--------|----------|
| **Cassandra / HBase** | Avoid disk lookups for SSTable files that don't contain the key |
| **Google Chrome** | Malicious URL filter — "Is this URL in the blocklist?" |
| **Bitcoin** | SPV clients filter transactions by address |
| **Redis (RedisBloom)** | Server-side Bloom filter |
| **Akamai CDN** | "Is this URL likely in the cache?" (avoid cache lookup for one-hit-wonders) |
| **Web crawlers** | "Have we visited this URL before?" (avoid re-crawling) |
| **Email spam filters** | Check sender against known-spam list |
| **Databases** | Check if a key exists before a disk read (RocksDB) |

### Cassandra Example

In Cassandra, each SSTable file on disk has an associated Bloom filter. Before reading from disk:
1. Bloom filter says "not present" → skip this SSTable (saves I/O)
2. Bloom filter says "probably present" → read from disk (may still not be there — false positive → wasted I/O)

With a 1% false positive rate, 99% of unnecessary disk reads are avoided.

---

## Q3: What are the limitations of Bloom filters?

1. **Cannot delete elements** — clearing a bit might remove evidence for another element
   - Fix: Use a **Counting Bloom Filter** (each position is a counter, not a bit; decrement on delete)

2. **No item enumeration** — you can't list what's in the filter

3. **Fixed capacity** — determined at creation time; can't resize (need to rebuild)
   - Fix: **Scalable Bloom Filter** — chain multiple filters together

4. **False positive rate grows** as elements are added beyond design capacity

---

## Q4: What is a Count-Min Sketch?

A Count-Min Sketch answers: **"How many times have I seen this element?"** (approximate frequency count)

### Structure

A 2D array with W columns and D rows. D independent hash functions.

**Increment element:**
```
For each row i (hash function hi):
  column = hi(element) % W
  array[i][column] += 1
```

**Query element count:**
```
For each row i:
  column = hi(element) % W
  candidate = array[i][column]
Return MIN of all candidates
```

**Why MIN?** Different elements may hash to the same cell (collisions add overcount). The minimum across rows gives the tightest upper bound.

### Use Cases

| System | Use Case |
|--------|----------|
| **Twitter** | Count tweet impressions, trending topic frequency |
| **Facebook** | Count user activity events |
| **Network routers** | Count packet frequencies (heavy hitter detection) |
| **Databases** | Query optimizer (estimate row counts for query planning) |
| **Advertising systems** | Count ad impressions per user without storing all events |

**Memory:** For error ε and confidence δ: W = e/ε, D = ln(1/δ)
Example: 1% error, 99% confidence → W=272, D=5. ~1360 counters vs potentially billions of actual items.

---

## Q5: What is HyperLogLog?

HyperLogLog answers: **"How many distinct elements have I seen?"** (cardinality estimation)

**Problem:** Counting unique visitors to a website. Naive: store every user ID seen → huge memory. HyperLogLog does it in ~12KB regardless of cardinality.

**How it works (simplified):**
- Hash each element
- Track the maximum number of leading zeros seen in any hash
- If max leading zeros = k, estimated cardinality ≈ 2^k
- Use multiple independent hash functions and take the harmonic mean for accuracy

**Accuracy:** ~0.81% standard error with 12KB memory (even for billions of distinct elements).

### Real-World Use

```
PFADD unique_visitors "user:123"
PFADD unique_visitors "user:456"
PFCOUNT unique_visitors → ~2 (approximate)
```

Redis has native HyperLogLog support. Used for:
- Counting unique daily active users
- A/B test reach (how many unique users saw variant A)
- Approximate COUNT(DISTINCT) in databases

---

## Q6: Comparison Summary

| Structure | Query | Error Type | Memory |
|-----------|-------|------------|--------|
| Bloom Filter | "Is X in set?" | False positives | ~10 bits/element |
| Count-Min Sketch | "How often is X?" | Overcount (not undercount) | ~1-5KB fixed |
| HyperLogLog | "How many distinct?" | ±0.81% cardinality | ~12KB fixed |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Bloom filter bits per element for 1% FP | ~10 bits |
| Bloom filter bits per element for 0.1% FP | ~15 bits |
| HyperLogLog memory (any cardinality) | ~12KB |
| HyperLogLog error rate | ~0.81% |
| Cassandra Bloom filter FP rate (default) | 1% (configurable) |

---

## Interview Q&A

**Q: Why does Cassandra use a Bloom filter per SSTable?**
A: Cassandra stores data in immutable SSTable files on disk. Without a Bloom filter, every read for a key that doesn't exist would require scanning every SSTable — expensive I/O. The Bloom filter lets Cassandra quickly say "this SSTable definitely doesn't have that key" for 99% of absent keys, reducing read latency dramatically.

**Q: Can you delete from a Bloom filter?**
A: Not from a basic Bloom filter — you can't safely clear a bit because it may have been set by another element. A Counting Bloom Filter stores a counter per position instead of a bit; deletion decrements the counter, and you remove the element when all counters reach 0. Trade-off: 4–8× more memory than a standard Bloom filter.

**Q: HyperLogLog vs exact count — when do you use each?**
A: Exact count requires storing every distinct element (or a hash set), which is O(n) memory. HyperLogLog uses O(1) memory with ~1% error — perfect for "how many unique users today?" where a 1% margin is fine. Use exact count only when you need precision (billing, fraud detection). For analytics and dashboards, HyperLogLog is the right choice.