# 41. DDoS Protection

## What is a DDoS Attack?

A **Distributed Denial of Service (DDoS)** attack floods a target with traffic from many sources (a botnet) with the goal of exhausting resources — bandwidth, CPU, memory, connection pools — so legitimate users can't be served.

**Why "distributed"?** Traffic comes from thousands/millions of compromised machines worldwide, making IP-based blocking ineffective and making it hard to distinguish from legitimate traffic spikes.

**Scale:** Modern DDoS attacks can exceed **1 Tbps** (terabits per second). For context, a typical web server has 1–10 Gbps of bandwidth.

---

## Q1: What are the types of DDoS attacks?

### Layer 3/4 — Volumetric (Network Layer)

Goal: Saturate bandwidth or exhaust stateful network resources.

| Attack | Mechanism | Size |
|--------|-----------|------|
| **UDP Flood** | Flood target with UDP packets to random ports | 100s of Gbps |
| **ICMP Flood (Ping Flood)** | Overwhelming ICMP echo requests | Gbps |
| **SYN Flood** | Send TCP SYN without completing handshake; fills server's SYN queue | Gbps |
| **DNS Amplification** | Send small DNS query spoofed as victim's IP → large response → victim flooded | 100s of Gbps |
| **NTP Amplification** | Similar to DNS; NTP monlist response is 200× the request size | Tbps scale |

**Amplification attacks:** Attacker sends a small packet to a reflector (DNS/NTP server) with the victim's IP as the source. The reflector sends a large response to the victim. Amplification ratio can be 50×–1000×.

```
Attacker (10 Mbps) → DNS servers (1000× amplification) → Victim (10 Gbps)
```

---

### Layer 7 — Application Layer

Goal: Exhaust server resources (CPU, DB connections, sessions) with seemingly legitimate requests.

| Attack | Mechanism |
|--------|-----------|
| **HTTP Flood** | Millions of valid HTTP GET/POST requests |
| **Slowloris** | Open many connections, send headers slowly — tie up web server thread pool |
| **Cache busting** | Send requests with unique query params (`?x=random`) to bypass cache, hit origin |
| **API abuse** | Hammer expensive endpoints (search, reports) |
| **Login brute force** | Mass login attempts (credential stuffing) |

**Why L7 is harder to mitigate:** Requests look like legitimate user traffic. You can't block them by IP (botnet uses thousands of IPs) or by packet pattern (valid HTTP).

---

## Q2: What is the DDoS protection stack?

Protection happens at multiple layers:

```
Internet
    │
    ▼
[CDN / Scrubbing Center]        ← Absorbs volumetric attacks (Cloudflare, Akamai)
    │
    ▼
[Anycast Network]               ← Distributes traffic across PoPs globally
    │
    ▼
[WAF (Web Application Firewall)] ← L7 filtering (rate limiting, bot detection)
    │
    ▼
[Load Balancer]                 ← Rate limiting, connection limits
    │
    ▼
[API Gateway]                   ← Per-key/per-IP rate limiting
    │
    ▼
[Application]
```

---

## Q3: How does Cloudflare protect against DDoS?

Cloudflare operates one of the world's largest anycast networks with **over 300 PoPs** and **>100 Tbps** of capacity — more than any known attack.

### Anycast routing

All Cloudflare PoPs announce the same IP ranges. Traffic is automatically routed to the nearest PoP via BGP. A 100 Gbps attack is distributed across all PoPs — no single PoP sees more than a fraction.

```
Attacker sends 100 Gbps to 104.16.0.0
BGP routes 30 Gbps → Frankfurt PoP
         20 Gbps → New York PoP
         25 Gbps → Singapore PoP
         25 Gbps → London PoP
Each PoP absorbs its share → origin server sees nothing
```

### L3/L4 mitigation

- **Magic Transit:** Cloudflare announces customer IP ranges → all traffic flows through Cloudflare → clean traffic tunneled back to customer
- **SYN cookies:** Respond to SYN with a cryptographic cookie in the SYN-ACK; no state allocated until ACK received — defeats SYN floods
- **Rate limiting at network level:** Drop UDP floods before they reach the application

### L7 mitigation — Bot Management

- **Browser integrity check:** Legitimate browsers execute JavaScript and set cookies. Bots often can't. Cloudflare challenges suspicious clients.
- **CAPTCHA / Managed Challenge:** For suspicious traffic, serve CAPTCHA — bots fail, humans pass.
- **Behavioral fingerprinting:** ML models trained on billions of requests to distinguish bots from humans (mouse movements, request timing, TLS fingerprints).
- **IP reputation:** Known botnet IPs, Tor exit nodes, data center IPs scored and challenged/blocked.
- **Rate limiting rules:**
```
Rule: Any IP making > 100 requests/min to /api/search → challenge
Rule: Any IP with > 5 failed logins/min → block for 1 hour
```

---

## Q4: What is a WAF (Web Application Firewall)?

A WAF inspects HTTP/HTTPS traffic at Layer 7 and blocks requests matching attack patterns.

### What WAF protects against
- **SQL injection:** `' OR 1=1 --` in query params
- **XSS (Cross-Site Scripting):** `<script>alert(1)</script>` in inputs
- **Path traversal:** `../../../../etc/passwd`
- **Remote file inclusion**
- **OWASP Top 10** vulnerabilities

### WAF rule types

**Signature-based:** Match known attack patterns (regexes). Fast, low false positives, but misses novel attacks.

**Anomaly-based:** Score each request; block if score exceeds threshold. Catches novel attacks but higher false positive risk.

**Rate-based:** Block IPs making too many requests in a time window.

### WAF options

| Option | Type | Notes |
|--------|------|-------|
| **Cloudflare WAF** | SaaS/CDN | Easy to enable, managed rulesets |
| **AWS WAF** | Cloud-managed | Integrates with CloudFront, ALB |
| **ModSecurity** | Open source | Self-hosted, OWASP Core Rule Set |
| **Imperva** | Commercial | Enterprise, advanced bot management |

---

## Q5: Rate Limiting as DDoS defense

Rate limiting at multiple levels:

### Global rate limiting (IP-based)
```
Nginx: limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
       limit_req zone=api burst=20 nodelay;
```
Block IPs exceeding 100 requests/minute. `burst=20` allows short bursts.

### Per-endpoint rate limiting
```
POST /login: 5 attempts/IP/minute → lock for 5 minutes
GET /search: 60 requests/IP/minute
GET /api/*: 1000 requests/API-key/minute
```

### Tiered rate limits
```
Free tier: 100 req/min
Pro tier: 10,000 req/min
Enterprise: 100,000 req/min
```

### Distributed rate limiting challenge
Single-server rate limiting doesn't work when you have 10 load-balanced servers — 100 req/min becomes 1000 req/min from the attacker's perspective.

**Solution:** Centralized rate limiting in Redis:
```lua
-- Redis Lua script (atomic)
local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window_seconds)
end
return current
```

All servers check the same Redis counter.

---

## Q6: Application-level DDoS defenses

### 1. Caching (reduces origin load)

If 90% of attack traffic hits cached responses at the CDN, your origin sees only 10%. Cache aggressively. Set long TTLs. Use cache-control headers.

**Cache busting defense:** Normalize query parameters before caching. `?x=random123` and `?x=random456` both map to the canonical URL.

### 2. Limit expensive operations

Search, report generation, data export are expensive. Rate limit them more aggressively. Queue expensive operations instead of serving synchronously.

### 3. CAPTCHA for suspicious clients

Challenge users who exhibit bot-like behavior (no cookies, no JS execution, high rate, known bot IP).

### 4. IP allowlisting for internal APIs

Admin endpoints, internal APIs — restrict to known IPs (office, VPN, other services). Block all other traffic at the firewall level.

### 5. Graceful degradation

Design the application to shed load under attack:
- Disable non-critical features (recommendations, analytics) while serving core functionality
- Return cached/stale results instead of real-time DB queries
- Implement circuit breakers to prevent DB exhaustion

---

## Q7: Incident response for a DDoS attack

```
T+0: Alert fires — traffic spike, error rate up
T+5: Identify attack type (L3/L4 vs L7, volumetric vs application)
T+10: Enable Cloudflare "Under Attack Mode" (JS challenge all visitors)
T+15: Analyze traffic — identify attack signature (user agent, URI, source country)
T+20: Create WAF rules to block attack pattern
T+30: Contact upstream provider if bandwidth saturation
T+60: Attack subsides or is fully mitigated
T+120: Post-mortem — improve defenses, update runbook
```

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Largest recorded DDoS | 3.47 Tbps (Microsoft Azure, 2021) |
| Cloudflare network capacity | >100 Tbps |
| DNS amplification ratio | 30–100× |
| NTP monlist amplification ratio | 200× |
| SYN flood mitigation (SYN cookies) | Eliminates state for incomplete handshakes |
| Typical botnet size | 100,000 – 1,000,000 compromised machines |
| Cloudflare PoP count | 300+ locations |
| HTTP flood typical rate | 100M–1B requests/hour |

---

## Interview Q&A

**Q: How would you design a system to be resilient against DDoS attacks?**
A: Defense in depth across layers. At the network layer: use a CDN/scrubbing service (Cloudflare, Akamai) as the first line — they absorb volumetric attacks before they reach your infrastructure. At the application layer: WAF for OWASP attacks, rate limiting per IP and per API key via Redis, CAPTCHA for suspicious clients. In the application: aggressive caching to reduce origin load, circuit breakers to prevent DB exhaustion, graceful degradation (serve stale data instead of crashing). At infrastructure: auto-scaling with conservative minimum capacity, separate rate limits for expensive endpoints like search.

**Q: What's the difference between a DDoS attack and a traffic spike from a viral post?**
A: Both look like sudden traffic increases, but the patterns differ. Legitimate viral traffic: geographically diverse, comes from real browsers (user agent variety, cookie support, JavaScript execution), follows normal browsing patterns (reads multiple pages, varied session lengths). DDoS: often from data center IPs, identical user agents, no JavaScript execution, hammer a single endpoint with no session behavior. Rate limiting helps with both, but WAF bot detection and behavioral fingerprinting differentiate them. Always check geography (is 90% from one country? suspicious), request diversity (same URL repeatedly?), and client characteristics (Googlebot fingerprint is different from a browser).

**Q: Why is L7 DDoS harder to defend against than L3/L4?**
A: L3/L4 attacks are distinguishable by packet characteristics — you can drop UDP floods or SYN floods at the network level before they cost you anything meaningful. L7 attacks use valid HTTP requests that are indistinguishable from legitimate traffic at the packet level. You must parse HTTP, understand the application, and make behavioral judgments. This means: you can't block at the network level (valid TCP connections), you spend more CPU per attack request (HTTP parsing), and you risk false positives (blocking real users). Defense requires ML-based bot detection, rate limiting with business logic, and CAPTCHA challenges — all of which introduce latency and complexity.