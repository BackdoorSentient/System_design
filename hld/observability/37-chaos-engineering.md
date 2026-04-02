# 37. Chaos Engineering

## What is Chaos Engineering?

Chaos Engineering is the discipline of **deliberately injecting failures into a production (or production-like) system** to discover weaknesses before they cause real incidents.

> "The best way to avoid failure is to fail constantly." — Netflix

The core idea: Your system will fail. Hardware dies, networks partition, services crash. It's better to discover failure modes through controlled experiments than to be surprised during peak traffic on Black Friday.

**From the Chaos Engineering Principles (principlesofchaos.org):**
> "Chaos Engineering is the discipline of experimenting on a system in order to build confidence in the system's capability to withstand turbulent conditions in production."

---

## Q1: Why chaos engineering instead of just testing?

**Traditional testing limitations:**
- Tests run in controlled environments (staging), which don't capture production complexity
- Tests verify known behavior — chaos finds *unknown* failure modes
- Staging rarely has the same scale, traffic patterns, or data as production
- Integration tests can't simulate "what if this one specific Cassandra node in us-east-1 gets network-partitioned at 2 AM?"

**Chaos engineering proactively answers:**
- Does our circuit breaker actually work when Service B goes down?
- Does failover complete in < 30 seconds when the primary DB crashes?
- Does our app degrade gracefully or does it return 500s when Redis is down?
- Does auto-scaling kick in within the SLO window?

---

## Q2: What is the scientific method of chaos engineering?

Chaos experiments follow a structured hypothesis-driven approach:

### 1. Define Steady State
Identify a measurable metric that represents normal system behavior.
- "P99 latency < 200ms for checkout endpoint"
- "Error rate < 0.1%"
- "95% of payments succeed within 3s"

### 2. Hypothesize
"We believe that X failure will NOT affect steady state because we have Y mitigation."
- "We believe killing one payment-service pod will not increase error rate because k8s will restart it in < 30s and our circuit breaker handles the interim."

### 3. Design Experiment
Define the failure to inject, the blast radius (scope), and a kill switch (how to stop).

### 4. Run Experiment
Start small — one pod, one region. Gradually expand.

### 5. Observe
Did steady state hold? If yes → increased confidence. If no → found a weakness → fix it.

### 6. Learn and Improve
Document findings. Fix the weakness. Expand blast radius.

---

## Q3: What is Chaos Monkey and the Simian Army?

**Chaos Monkey** was Netflix's original chaos tool (2011). It randomly terminates EC2 instances in production. The idea: since any instance can die, engineers must build services that survive instance death — or they'd be constantly paged at 3 AM.

**The Simian Army** — Netflix's suite of chaos tools:

| Tool | What it does |
|------|-------------|
| **Chaos Monkey** | Randomly kills EC2 instances |
| **Chaos Gorilla** | Kills an entire AWS availability zone |
| **Chaos Kong** | Simulates an entire AWS region failure |
| **Latency Monkey** | Injects artificial network latency |
| **Doctor Monkey** | Detects unhealthy instances and removes them |
| **Janitor Monkey** | Cleans up unused cloud resources |
| **Conformity Monkey** | Checks if instances follow best practices |
| **Security Monkey** | Finds security policy violations |

Netflix ran Chaos Monkey in production 24/7 — even during business hours. The logic: if you only run chaos on nights and weekends, engineers will design systems that work during nights and weekends. Business-hours chaos forces you to build systems robust enough to handle failures when customers are actively using them.

---

## Q4: What types of failures can chaos engineering inject?

### Infrastructure failures
- Kill a pod/container/VM
- Kill an entire availability zone
- Simulate node out-of-memory (OOM)
- Fill disk to 100%

### Network failures
- Add artificial latency (e.g., +300ms to all calls to Service B)
- Packet loss (e.g., 10% of packets dropped)
- Partition network between services
- DNS failure (block DNS resolution)

### Application failures
- Kill a specific process (e.g., the leader in a Raft cluster)
- Inject exceptions at certain code paths
- Corrupt data (return malformed responses)
- Slow responses (inject sleep before responding)

### Resource exhaustion
- CPU spike (run CPU-intensive workloads)
- Memory leak simulation
- Exhaust file descriptors
- Exhaust DB connection pool

### Dependency failures
- Make a third-party API return 500s
- Slow down Redis to 2s response time
- Make S3 return 503s

---

## Q5: What are blast radius and kill switches?

### Blast Radius
The scope of the experiment. Always start small:

```
Level 1: One pod → fails gracefully, k8s restarts it
Level 2: All pods of one service → circuit breakers engage
Level 3: One availability zone → cross-AZ failover activates
Level 4: One region → global failover + disaster recovery
```

Never jump to Level 4 without confidence at Level 1.

**Blast radius controls:**
- Run chaos only on a % of traffic (10% of requests experience failure)
- Run chaos only in one AZ
- Run chaos only for non-VIP users
- Run chaos only during off-peak hours (initially)

### Kill Switch
An immediate way to stop the experiment if things go wrong.

- A feature flag that disables chaos injection instantly
- A hard stop time (experiment auto-terminates after 10 minutes)
- On-call engineer with a "break glass" procedure
- Automatic abort if steady-state SLO is breached beyond a threshold

---

## Q6: What is a Game Day?

A **Game Day** is a scheduled, cross-team chaos exercise where engineers simulate a specific disaster scenario and practice the incident response.

**Structure:**
1. **Scenario:** "The primary database in us-east-1 is unavailable"
2. **Participants:** SRE, dev team leads, on-call engineers
3. **Environment:** Staging or a isolated production slice
4. **Injects:** Simulate the failure, observe responses
5. **Debrief:** What worked? What didn't? What would have happened in prod?

**Value beyond technical:** Game days also test:
- Runbook clarity (can engineers follow them under stress?)
- Alert quality (did the right alerts fire?)
- Communication (did the team know what was happening?)
- Decision-making (did engineers know when to escalate?)

---

## Q7: Chaos Engineering tools

| Tool | By | Features |
|------|----|----|
| **Chaos Monkey** | Netflix | Instance termination on AWS |
| **Chaos Toolkit** | ChaosIQ | Open source, extensible, k8s + cloud |
| **LitmusChaos** | CNCF | Kubernetes-native, rich fault library |
| **Gremlin** | Gremlin Inc. | Commercial, user-friendly UI, many fault types |
| **AWS Fault Injection Simulator (FIS)** | AWS | Managed AWS chaos, native integrations |
| **Chaos Mesh** | PingCAP | Kubernetes-native, visual dashboard |
| **Toxiproxy** | Shopify | Network fault injection (latency, packet loss) for dev/test |

---

## Q8: What should you chaos-test first?

Prioritize based on **impact × likelihood**:

1. **Single point of failure identification** — kill each component and see what happens
2. **Failover validation** — does auto-failover actually work and meet SLO?
3. **Dependency failures** — what happens when each external dependency fails?
4. **Traffic overload** — does autoscaling kick in before SLO breach?
5. **Data corruption resilience** — what happens if a service returns bad data?

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Netflix runs Chaos Monkey | Continuously, 24/7 in production |
| Typical blast radius for first experiment | 1 pod, 5–10% traffic |
| Game day frequency (mature orgs) | Quarterly |
| Time to kill switch activation | Should be < 60 seconds |
| AWS FIS typical experiment duration | 5–30 minutes |

---

## Interview Q&A

**Q: Isn't running chaos in production reckless? How do you justify it to leadership?**
A: The alternative is being surprised by failures during peak traffic with no practice. Controlled chaos with a defined blast radius, kill switches, and observability is far safer than discovering failure modes during a real incident at 2 AM. Netflix's argument: production failures happen regardless; the question is whether you encounter them for the first time during a chaos experiment (controlled) or during an actual outage (uncontrolled). Start with non-critical services, low blast radius, and off-peak hours to build confidence before expanding scope.

**Q: How is chaos engineering different from load testing?**
A: Load testing validates behavior under high traffic — it answers "how many requests can we handle?" Chaos engineering validates behavior under failures — it answers "what happens when something breaks?" They're complementary. Load test first to understand capacity. Chaos test to understand resilience. You could also combine them: run chaos experiments *under* peak load to test failure handling in the worst case.

**Q: You run a chaos experiment and your circuit breakers don't trip. What went wrong?**
A: Either (1) the circuit breaker thresholds are set too high (needs more failures to trip), (2) the circuit breaker is misconfigured (wrong service, wrong method intercepted), (3) the failure mode you injected isn't one the circuit breaker monitors (e.g., you injected slow responses but the circuit breaker only monitors error status codes), or (4) the circuit breaker library isn't actually wired into the code path. Chaos experiments expose exactly this kind of misconfiguration — the circuit breaker gives you false confidence if you never test that it actually fires.