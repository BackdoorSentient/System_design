# 33. Alerting Strategies

## What Makes a Good Alert?

An alert should be **actionable, urgent, and novel**. It should wake someone up only when a human needs to do something *right now* that can't be automated.

**Alert fatigue** — the most dangerous anti-pattern in alerting. When teams receive too many alerts (especially false positives), they start ignoring them. The alert that matters gets missed. Studies show that teams with high alert fatigue miss ~30% of critical incidents.

---

## Q1: What are the four golden signals for alerting?

From the Google SRE book — these four signals cover the most important aspects of service health:

| Signal | What to Alert On | Example Threshold |
|--------|-----------------|-------------------|
| **Latency** | P99 response time exceeds SLO | P99 > 1s for 5 minutes |
| **Traffic** | Sudden drop (likely an upstream failure) | RPS drops >50% in 5 min |
| **Errors** | Error rate exceeds threshold | Error rate > 1% for 5 min |
| **Saturation** | Resource approaching capacity | CPU > 85%, queue depth > 1000 |

---

## Q2: What is symptom-based vs cause-based alerting?

### Cause-based alerting (bad default)
Alerting on the *internal state* of the system:
- "CPU usage > 80%"
- "Disk I/O wait > 50ms"
- "GC pause > 200ms"

**Problem:** These might not affect users at all. Or there might be 10 different causes for user impact, and you'd need 10 alerts to cover them all. Cause-based alerts create noise and miss novel failure modes.

### Symptom-based alerting (preferred)
Alerting on *user-visible impact*:
- "Error rate > 1%"
- "P99 latency > 2 seconds"
- "Payment success rate < 99%"

**Principle:** Alert on what users experience. Use metrics/logs/traces to diagnose *why* after the alert fires.

**The combination:**
- Page on symptoms (wake someone up for user impact)
- Ticket/notify on causes (CPU high, disk low) for awareness without paging

---

## Q3: What is SLO-based alerting?

**SLO (Service Level Objective):** A target for service reliability. E.g., "99.9% of requests will succeed" or "P99 latency < 500ms over 30 days."

**Error budget:** The allowed amount of failure before the SLO is breached. For 99.9% over 30 days: `0.1% × 30 days × 24h × 3600s = 2,592 seconds` of downtime allowed.

**SLO-based alerting** (aka burn rate alerts) alerts when you're **consuming your error budget too fast**.

### Burn Rate

If your error budget is 2,592 seconds over 30 days:
- Normal burn rate = 1× (consuming budget exactly at the expected rate)
- Burn rate = 14× = consuming budget 14× faster than normal → budget runs out in ~50 hours

**Multi-window, multi-burn-rate alerts (Google's recommendation):**

| Urgency | Burn Rate | Time Window | Response |
|---------|-----------|-------------|----------|
| 🔴 Page (critical) | 14× | 1h + 5min | Wake up now — will exhaust budget in ~2 days |
| 🟠 Page (high) | 6× | 6h + 30min | Investigate urgently — budget gone in ~5 days |
| 🟡 Ticket | 3× | 24h + 6h | Fix this sprint — budget gone in ~10 days |
| 🟢 None | 1× | — | Normal operation |

**Why two windows?** Short window catches fast burns quickly. Long window prevents false positives from brief spikes. Both must be true simultaneously to alert.

---

## Q4: How do you structure alerts to avoid alert fatigue?

### 1. Severity tiers

| Tier | Response | Example |
|------|----------|---------|
| **P0 — Critical** | Page on-call immediately, 24/7 | Payment service down |
| **P1 — High** | Page during business hours | Error rate >5% on non-critical service |
| **P2 — Medium** | Slack notification, fix next day | Disk usage >80% |
| **P3 — Low** | Ticket created | Deprecated API still in use |

### 2. Alert routing
Route different alerts to different channels:
- P0 → PagerDuty → phone call
- P1 → PagerDuty → SMS
- P2 → Slack #alerts-prod
- P3 → Jira ticket auto-created

### 3. Grouping and deduplication
If 100 pods all report the same error, fire **one alert** with count=100, not 100 separate alerts. Alertmanager (Prometheus) and PagerDuty both support grouping.

### 4. Inhibition rules
If the database is down, suppress all alerts that are downstream effects of the database being down. Don't alert "payment service errors" when "DB connectivity down" already covers it.

### 5. Maintenance windows
Suppress alerts during planned maintenance to avoid noise.

---

## Q5: What makes a good alert definition?

Every alert should answer these questions in its definition:

```yaml
# Example Prometheus alert rule
- alert: PaymentServiceHighErrorRate
  expr: |
    rate(http_requests_total{service="payment",status=~"5.."}[5m])
    /
    rate(http_requests_total{service="payment"}[5m])
    > 0.01
  for: 5m                    # Must be true for 5 consecutive minutes (avoid flapping)
  labels:
    severity: critical
    team: payments
  annotations:
    summary: "Payment service error rate above 1%"
    description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
    runbook_url: "https://wiki.company.com/runbooks/payment-service-errors"
    dashboard_url: "https://grafana/d/payment-service"
```

**The `for` clause:** Prevents alerts from firing on brief spikes. 5 minutes of sustained error rate is much more meaningful than 30 seconds.

---

## Q6: What is a runbook?

A **runbook** (also called a playbook) is step-by-step documentation for responding to a specific alert. Every page-worthy alert should have a runbook linked in its annotation.

**Runbook contents:**
1. What does this alert mean? (plain English)
2. What is the user impact?
3. Immediate triage steps (first 5 minutes)
4. Common causes and how to verify each
5. Remediation steps for each cause
6. Escalation path if you can't fix it

**Why runbooks matter:** They allow junior engineers to handle incidents. They reduce mean time to resolution (MTTR). They prevent institutional knowledge from living only in senior engineers' heads.

---

## Q7: Alertmanager — the Prometheus alerting layer

**Prometheus** evaluates alert rules. **Alertmanager** handles routing, deduplication, silencing, and notification.

```
Prometheus → [fires alert] → Alertmanager
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼              ▼
               PagerDuty       Slack          Email
```

**Key Alertmanager features:**
- **Grouping:** Combine related alerts into one notification
- **Inhibition:** Suppress child alerts when parent fires
- **Silences:** Mute alerts during maintenance
- **Routing:** Different receivers for different labels

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Alert fatigue threshold | >5–10 pages/shift = team starts ignoring |
| SLO breach burn rate (Google recommendation) | Alert at 14× (1h window) and 6× (6h window) |
| Typical alert evaluation interval | 1 minute |
| `for` clause minimum (avoid flapping) | 5 minutes |
| MTTA (Mean Time to Acknowledge) target | <5 minutes for P0 |
| MTTR (Mean Time to Resolve) target | <30 min for P0, <4h for P1 |

---

## Real-World Examples

| Company | Approach |
|---------|---------|
| Google | SRE book defines multi-burn-rate SLO alerts |
| Netflix | Symptom-based alerting, chaos engineering to validate alert coverage |
| Airbnb | Alert budget per team — limits number of pages per week |
| Stripe | Tiered severity with explicit escalation paths |

---

## Interview Q&A

**Q: Your team gets 50 pages per shift and engineers are burning out. How do you fix this?**
A: Alert audit — classify every alert from the last month as: (1) actionable and correctly fired, (2) too noisy / false positive, (3) duplicate of another alert. For category 2: add `for` clauses to avoid flapping, raise thresholds, or convert to non-paging notifications. For category 3: add inhibition rules. Move to SLO-based alerting — fewer, higher-quality alerts tied directly to user impact. Establish a weekly alert review meeting to continuously prune noise.

**Q: What's the difference between an SLA, SLO, and SLI?**
A: SLI (Service Level Indicator) is the actual measurement — "our error rate is 0.05%." SLO (Service Level Objective) is the target we set for ourselves — "error rate should be < 0.1%." SLA (Service Level Agreement) is the contractual commitment to customers — "if error rate exceeds 0.5%, customers get a refund." SLOs are typically stricter than SLAs to give an internal buffer. You alert on SLO breach before you breach the SLA.

**Q: Why use burn rate alerting instead of a simple threshold like "error rate > 1%"?**
A: A fixed threshold doesn't account for time. An error rate of 5% for 30 seconds is very different from 5% for 6 hours. Burn rate ties the alert to your error budget — it answers "at this rate, will we breach our monthly SLO?" A 1% steady error rate might be perfectly acceptable if your SLO is 99.9% (you'd still meet it). But a 10% error rate for even a few hours would consume your entire monthly budget. Burn rate makes this explicit and proportional.