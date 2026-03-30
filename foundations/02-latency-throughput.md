# Latency vs Throughput

**Q: What is latency?**
A: Latency is the total time from when a client sends a request to when it receives the complete
response. It is measured in milliseconds (ms) and has several components:

- Network latency: Time for packets to travel across the network (affected by physical distance
  and routing hops). A request from Mumbai to a server in Virginia adds ~200ms round-trip.
- Processing latency: Time the server spends computing the response — querying DB, running
  business logic, rendering templates.
- Queuing latency: Time the request spends waiting in a queue before a worker picks it up.
  Under high load, this dominates.
- Serialization latency: Time to encode/decode data (JSON parsing, protobuf serialization).

Total latency = network + queue wait + processing + network back.

**Q: What is throughput?**
A: Throughput is the number of requests a system successfully handles per unit of time. Measured
as QPS (queries per second), RPS (requests per second), or TPS (transactions per second).

Throughput is a capacity measure — it tells you the ceiling of your system under sustained load.
A system with 10,000 QPS throughput can handle 10,000 requests every second continuously.

**Q: What is the relationship between latency and throughput?**
A: They are related but distinct, and optimizing one can hurt the other:

- At low load: latency is low, throughput is limited by request rate.
- As load increases: throughput increases, but latency starts rising as queues form.
- At saturation: throughput plateaus (you've hit your ceiling) and latency spikes because
  requests are waiting in long queues.

Little's Law formalizes this: L = λW, where L = average number of requests in the system,
λ = throughput (arrival rate), W = average latency. If throughput doubles and latency doubles,
you have 4× as many concurrent requests in-flight.

**Q: What are percentile latencies and why are they more useful than averages?**
A: Percentile latencies sort all response times and report the value at a given percentile:

- P50 (median): 50% of requests are faster than this. Represents the typical user.
- P90: 90% of requests are faster. Starting to capture slower experiences.
- P99: 99% of requests are faster. Your "worst common case" — 1 in 100 users experiences this.
- P999: 99.9% are faster. Rare outliers — usually caused by GC pauses, cold cache, or DB locks.

Why averages mislead: If 99% of requests take 50ms and 1% take 10,000ms, the average might be
150ms — which sounds fine but hides the fact that 1 in 100 users is waiting 10 seconds. P99
would expose this immediately.

In interviews and production monitoring, always design to a P99 target, not an average.

**Q: What causes latency spikes at the P99/P999 level?**
A: Common causes:
- Garbage collection pauses (JVM, Go): GC can stop the world for 100ms–2s, blocking all
  requests being processed during that window.
- Lock contention: Multiple threads competing for the same database row or mutex.
- Cold cache misses: The first request after a cache eviction hits the database directly.
- Connection pool exhaustion: All DB connections are in use; new requests queue up.
- Network jitter: Packet loss causes TCP retransmission, adding 1–3× the round-trip time.
- Disk I/O: A slow disk read can block a thread for hundreds of milliseconds.
- Head-of-line blocking: In HTTP/1.1, one slow request blocks all subsequent requests on
  the same connection.

**Q: What is a bottleneck and how do you find one?**
A: A bottleneck is the single resource that is 100% saturated while everything else is idle —
it limits the entire system's throughput. Every system has exactly one bottleneck at any time
(the Theory of Constraints).

How to find it:
1. Measure CPU utilization across all services under load.
2. Measure database query times and connection pool wait times.
3. Measure network bandwidth and packet loss.
4. Measure disk I/O wait.
5. Look at queue depths — a growing queue upstream of a component means that component is
   the bottleneck.

Tools: Flame graphs (find hot CPU paths), slow query logs (find DB bottlenecks), distributed
tracing (Jaeger, Zipkin — find which service adds the most latency end-to-end).

Fix the bottleneck, then re-measure. The next bottleneck will reveal itself.

**Q: What is tail latency amplification in microservices?**
A: When a request fans out to multiple services, the total latency is the MAX of all service
latencies, not the average. This means tail latency gets amplified.

Example: A request calls 10 services in parallel. Each has P99 of 100ms. The probability that
at least one of the 10 calls hits its P99 is 1 - (0.99)^10 = ~10%. So your end-to-end P99 is
much worse than any individual service's P99.

At Google scale, with hundreds of parallel calls per request, tail latency is a constant battle.
Solutions: hedged requests (send the same request to two servers, use the first response),
timeouts, and circuit breakers.

**Q: What is the difference between latency-sensitive and throughput-sensitive workloads?**
A: Latency-sensitive (online) workloads: User is waiting for the response. Every millisecond
matters. Examples: web page loads, API calls, database queries, payment processing. Optimize
for P99 latency.

Throughput-sensitive (offline/batch) workloads: No user is waiting. You want to process as
much data as possible. Examples: ETL pipelines, report generation, ML training, log processing.
Optimize for total data processed per hour, not latency of individual operations.

**Q: What is queueing theory and why does it matter?**
A: Queueing theory models how requests behave when they arrive faster than they're processed.
The key insight from the Erlang-C model: as utilization approaches 100%, queue length (and
therefore latency) approaches infinity. This is why systems feel fine at 70% utilization and
collapse at 95% — the queue builds exponentially near saturation.

Practical rule: Keep server utilization below 70–80% for interactive workloads. The last 20%
of capacity produces exponential latency growth, not linear.