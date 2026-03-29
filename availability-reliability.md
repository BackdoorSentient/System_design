# Availability, Reliability & Fault Tolerance

**Q: What is availability and how is it measured?**
A: Availability is the fraction of time a system is operational and able to serve requests.
Measured as a percentage over a rolling window (usually monthly or yearly).

Formula: Availability = Uptime / (Uptime + Downtime)

The "nines":
| Availability   | Downtime/year  | Downtime/month | Downtime/week |
|----------------|---------------|----------------|---------------|
| 99% (two 9s)   | 3.65 days     | 7.2 hours      | 1.68 hours    |
| 99.9% (three)  | 8.76 hours    | 43.8 minutes   | 10.1 minutes  |
| 99.99% (four)  | 52.6 minutes  | 4.4 minutes    | 1.01 minutes  |
| 99.999% (five) | 5.26 minutes  | 26.3 seconds   | 6.05 seconds  |

Going from three to four nines sounds small (0.09%) but cuts downtime from 8.76 hours to
52 minutes per year — a 10× improvement that requires fundamentally different architecture.

**Q: What is the difference between availability, reliability, and fault tolerance?**
A: These are related but distinct:

Availability: Is the system responding right now? A percentage measured over time.
A system that crashes for 5 minutes every day is 99.65% available.

Reliability: Does the system produce correct results consistently? A system can be available
(online) but unreliable (returning corrupted data, wrong calculations, or errors). Reliability
is about correctness and consistency over time. MTBF (Mean Time Between Failures) measures it.

Fault tolerance: Can the system keep working despite component failures? A fault-tolerant
system has redundancy — if one component fails, another takes over automatically with no
visible impact. Fault tolerance is the mechanism that achieves high availability.

Example: A bank's ATM network — available (mostly online), reliable (correct balances),
fault tolerant (one ATM failing doesn't affect others).

**Q: What is MTTR and MTBF and how do they relate to availability?**
A: MTBF (Mean Time Between Failures): Average time a system runs before failing.
MTTR (Mean Time To Recover): Average time to restore service after a failure.

Availability = MTBF / (MTBF + MTTR)

To increase availability you can either:
- Increase MTBF: Make the system fail less often (better hardware, more testing, redundancy).
- Decrease MTTR: Recover faster when failures happen (automated failover, good runbooks,
  on-call processes, feature flags to roll back quickly).

Most high-availability engineering is actually about reducing MTTR — failures are inevitable,
fast recovery is what keeps your SLA intact.

**Q: What is an SLA, SLO, and SLI — and how do they differ?**
A: These are three levels of availability commitment:

SLI (Service Level Indicator): The actual measured metric. Examples: request success rate,
P99 latency, error rate, uptime percentage. This is what you instrument and observe.

SLO (Service Level Objective): Your internal target for an SLI. Example: "P99 latency must
stay below 200ms" or "Error rate must stay below 0.1%". This is your internal engineering
goal. SLOs are typically stricter than your SLA to give yourself a buffer.

SLA (Service Level Agreement): A contract with customers (or between teams) promising a
certain level of service. Violating it has consequences — refunds, credits, penalties.
Example: AWS promises 99.99% uptime for EC2. If they miss it, you get service credits.

The hierarchy: SLA (external promise) > SLO (internal target) > SLI (measured reality).
If your SLI shows you're trending toward your SLO limit, you fix it before breaching the SLA.

**Q: What is an error budget?**
A: An error budget is the amount of downtime or errors you're allowed per SLO period. If your
SLO is 99.9% monthly, your error budget is 0.1% of a month = 43.8 minutes of downtime.

This budget is shared between engineering teams (their deploys can cause outages) and ops
(infrastructure failures). When the error budget is nearly exhausted, you freeze new feature
releases and focus on reliability. When the budget is healthy, you can move fast.

Error budgets make reliability a business conversation, not just an engineering one. Product
managers can't demand 100% uptime AND fast releases — the error budget forces an explicit
trade-off discussion.

**Q: What design patterns achieve high availability?**
A: Key patterns:

Redundancy: Run N+1 or N+2 instances of every component. If one fails, others absorb the
traffic. Apply at every layer: web servers, app servers, databases, load balancers, even
entire data centers.

Active-active vs active-passive: Active-active means all instances serve traffic simultaneously
(better utilization). Active-passive means standby instances take over only on failure (simpler
but wastes capacity).

Automatic failover: Health checks detect failures and traffic is rerouted within seconds —
no human intervention required. Manual failover means downtime equals however long it takes
your on-call engineer to wake up and respond.

Geographic distribution: Deploy in multiple regions/availability zones. AWS has 3+ AZs per
region, isolated from each other's power and networking. A failure in us-east-1a doesn't
affect us-east-1b.

Graceful degradation: When a dependency fails, serve a degraded response rather than an
error. Example: If the recommendation service is down, show popular items instead of
personalized ones. The page still loads; it's just less optimal.

Circuit breaker: If a downstream service starts failing, stop sending it requests (open the
circuit) and return a fallback immediately. This prevents cascading failures where one slow
service takes down everything upstream.

**Q: What is a cascading failure and how do you prevent it?**
A: A cascading failure occurs when one component's failure causes overload in other components,
which then fail, causing further overload — a domino effect that brings down the whole system.

Example: Database slows down → API server threads pile up waiting for DB → API server runs
out of threads → load balancer health checks fail → more retries from clients → even more
load on the DB → total outage.

Prevention strategies:
- Circuit breakers: Stop calling a failing service, fail fast instead of piling up.
- Timeouts everywhere: Never wait indefinitely. A 30-second timeout stops threads from
  piling up.
- Bulkheads: Isolate thread pools per downstream dependency. DB thread pool exhaustion
  doesn't affect the cache thread pool.
- Rate limiting and load shedding: Reject excess requests early (return 429 Too Many
  Requests) rather than accepting everything and collapsing.
- Backpressure: Signal upstream components to slow down when you're overwhelmed.

**Q: What is chaos engineering?**
A: Chaos engineering is the practice of intentionally injecting failures into a production
system to verify that it handles them gracefully. Pioneered by Netflix (Chaos Monkey).

Examples of what you inject: killing random servers, introducing network latency, corrupting
packets, failing a database, draining an entire availability zone.

The goal is to find weaknesses before a real failure does. If your system survives planned
chaos, you have confidence it will survive unplanned failures. It also forces teams to build
proper monitoring, alerting, and runbooks.