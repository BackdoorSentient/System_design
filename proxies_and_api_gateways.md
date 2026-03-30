# 08_proxies_and_api_gateways.md — Proxies & API Gateways

---

**Q: What is the difference between a forward proxy and a reverse proxy?**

**Forward Proxy**:
- Sits between clients and the internet, acting on behalf of the **client**.
- The client is configured to route requests through the proxy. The destination server sees the proxy's IP, not the client's.
- Use cases:
  - Corporate internet filtering (block social media, log all traffic).
  - Bypassing geo-restrictions (VPN-like behavior).
  - Client anonymization.
  - Caching outbound requests (save bandwidth in large offices).
- Examples: Squid, corporate proxy servers, VPN endpoints.

**Reverse Proxy**:
- Sits in front of backend servers, acting on behalf of the **server**.
- Clients talk to the reverse proxy as if it were the server. The client doesn't know about the backend topology.
- Use cases:
  - Load balancing across backend instances.
  - SSL/TLS termination (offload from app servers).
  - Caching responses.
  - Compression (gzip/brotli).
  - Static file serving.
  - Security (WAF, rate limiting, DDoS protection).
- Examples: Nginx, HAProxy, Caddy, AWS ALB, Cloudflare.

**Memory aid**: Forward = on behalf of the client. Reverse = on behalf of the server.

---

**Q: What is Nginx and what are its primary use cases in production?**

Nginx is a high-performance, event-driven HTTP server and reverse proxy. Originally designed to solve the C10K problem (handling 10,000 concurrent connections with a single server).

**Architecture**: Unlike Apache's process-per-request model, Nginx uses an asynchronous, event-driven (non-blocking) architecture. A small number of worker processes (typically equal to CPU count) handle thousands of connections using an event loop.

**Primary use cases**:

**1. Reverse proxy / load balancer**:
```nginx
upstream backend {
    server app1:8080 weight=3;
    server app2:8080 weight=1;
    keepalive 64;  # persistent connections to backend
}

server {
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**2. SSL termination**:
```nginx
server {
    listen 443 ssl http2;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
}
```

**3. Static file serving**: Nginx can serve static files at extremely high throughput (serving from OS page cache with sendfile syscall — zero-copy). Much more efficient than serving static files from a Node.js or Python app server.

**4. Caching**: Nginx can cache upstream responses on disk. Cache keys are configurable (URL, headers). Useful for expensive, cacheable endpoints.

**5. Compression**: `gzip on;` — Nginx compresses responses before sending. Typically reduces JSON/HTML payload by 60–80%.

**6. Rate limiting**:
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
}
```

**Performance**: Nginx can handle ~50,000–100,000 connections per worker process and serve ~100,000+ static file requests/sec on modern hardware.

---

**Q: What is an API Gateway and how does it differ from a reverse proxy?**

A **reverse proxy** is a general-purpose component that forwards requests, terminates SSL, and load balances. It operates at the HTTP layer but is not aware of API semantics.

An **API Gateway** is a specialized reverse proxy built specifically for managing APIs. It adds:

| Feature | Reverse Proxy | API Gateway |
|---|---|---|
| SSL termination | Yes | Yes |
| Load balancing | Yes | Yes |
| Rate limiting | Basic | Advanced (per-user, per-plan, per-endpoint) |
| Authentication | No | Yes (JWT validation, OAuth, API keys) |
| Request/response transformation | No | Yes (add/remove headers, body transformation) |
| API versioning routing | No | Yes |
| Logging / analytics | Basic | Detailed API usage analytics |
| Developer portal | No | Often included |
| Plugin/middleware system | No | Yes |
| Circuit breaking | No | Yes |
| Caching | Basic | Advanced |

**Examples**:
- **Kong**: Open-source, plugin-based. Can run on Nginx or OpenResty. Highly extensible.
- **AWS API Gateway**: Fully managed. Integrates natively with Lambda, IAM, Cognito.
- **Apigee** (Google): Enterprise-grade with strong analytics and developer portal.
- **Traefik**: Cloud-native, auto-discovers services in Kubernetes/Docker.
- **Envoy**: High-performance proxy used in service meshes (Istio). Highly configurable.

---

**Q: How does Kong work and what are its key features?**

Kong is an open-source API gateway built on top of Nginx/OpenResty (Nginx + LuaJIT runtime). It's plugin-based — core functionality is extended via plugins.

**Architecture**:
```
Client → Kong (Gateway) → Backend Services
             ↓
         Database (PostgreSQL or Cassandra for state)
         or DB-less mode (declarative YAML config)
```

**Core entities**:
- **Service**: Represents an upstream API (e.g., `users-service` → `http://users:8080`).
- **Route**: A rule matching incoming requests to a Service (e.g., `Host: api.example.com, Path: /users`).
- **Plugin**: Applied to a Service, Route, or Consumer. Controls behavior.
- **Consumer**: Represents an API user/application. Associates with credentials.

**Key plugins**:
- `key-auth`: Validates API keys. Associates requests with a Consumer.
- `jwt`: Validates and decodes JWT tokens.
- `rate-limiting`: Per-consumer or per-IP rate limits. Redis-backed for distributed enforcement.
- `proxy-cache`: Caches upstream responses.
- `request-transformer`: Adds/removes/modifies request headers and body.
- `response-transformer`: Modifies responses.
- `cors`: Adds CORS headers.
- `ip-restriction`: Whitelist/blacklist IP ranges.
- `log`: Ships access logs to HTTP, TCP, Kafka, Datadog, Splunk.
- `circuit-breaker`: Prevents cascading failures.

**Declarative config (DB-less mode)**:
```yaml
_format_version: "3.0"
services:
  - name: users-service
    url: http://users-backend:8080
    routes:
      - name: users-route
        paths: ["/api/users"]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
```

**Kong Mesh / Konnect**: Enterprise layer adding service mesh, multi-cluster management, and SaaS control plane.

---

**Q: What is a service mesh and how does it relate to API gateways?**

A **service mesh** (Istio, Linkerd, Consul Connect) manages service-to-service (east-west) traffic within a cluster. An **API gateway** manages external (north-south) traffic into the cluster.

**How a service mesh works**:
- Deploys a **sidecar proxy** (Envoy in Istio's case) alongside every microservice pod.
- All traffic in/out of a pod goes through the sidecar, not directly.
- The control plane (Istio's Istiod) configures all sidecar proxies centrally.

**What the sidecar provides**:
- Mutual TLS (mTLS) between all services — encryption and authentication for service-to-service calls without code changes.
- Load balancing with circuit breaking.
- Distributed tracing (Jaeger, Zipkin).
- Retry logic with backoff.
- Traffic shaping: canary deployments (5% of traffic to v2, 95% to v1).
- Observability: L7 metrics for every service-to-service call.

**API Gateway vs Service Mesh**:
- API Gateway: North-south. External client → cluster. Authentication, rate limiting, routing.
- Service Mesh: East-west. Service A → Service B inside the cluster. mTLS, observability, resilience.
- They complement each other. You typically have both.

---

**Q: What is SSL/TLS termination and why do you terminate at the proxy rather than the app?**

**SSL/TLS termination** is the process of decrypting TLS-encrypted traffic at the proxy/load balancer level and forwarding plain HTTP to backend servers.

**Why terminate at the proxy**:
1. **Performance**: TLS handshakes and cryptographic operations are CPU-intensive. Offloading to the proxy frees app servers for business logic. Proxies can use hardware crypto accelerators.
2. **Centralized certificate management**: One certificate and renewal process at the proxy rather than on every app server.
3. **Simpler backends**: App servers receive plain HTTP; no TLS library required in application code.
4. **Session resumption**: The proxy handles TLS session tickets and resumption, reducing handshake overhead for reconnecting clients.

**Security consideration**: Traffic between the proxy and backends is unencrypted (HTTP). This is usually acceptable within a private VPC/data center network. For compliance-sensitive environments (PCI-DSS, HIPAA), re-encrypt with TLS to the backends (TLS passthrough or re-encryption mode).

**TLS passthrough**: The proxy forwards encrypted traffic directly to the backend without decrypting. The backend handles TLS. Used when you need end-to-end encryption or when the backend requires client certificates. The proxy can't inspect or modify the request in this mode.

---

**Q: What is a circuit breaker pattern and how is it implemented in a proxy/gateway?**

**Problem**: Service A calls Service B. Service B is slow (high latency). Service A's thread pool fills with waiting calls. Service A becomes slow for all its users, even for requests that don't involve Service B. Cascading failure.

**Circuit Breaker states**:
- **Closed** (normal): Requests pass through. Failure rate is monitored.
- **Open** (tripped): Failure rate exceeded threshold. All requests to Service B are immediately rejected (fail fast) without attempting the call. After a timeout (e.g., 30 seconds), transitions to Half-Open.
- **Half-Open**: Allow a small number of test requests through. If they succeed, close the circuit. If they fail, re-open.

**Parameters**:
- `failure_threshold`: e.g., 50% error rate over last 10 requests.
- `open_timeout`: How long to stay open before trying again (e.g., 30 seconds).
- `half_open_max_requests`: How many test requests to allow in Half-Open state.

**Implementation in Envoy/Istio**:
```yaml
trafficPolicy:
  outlierDetection:
    consecutive5xxErrors: 5        # Trip after 5 consecutive 5xx errors
    interval: 30s                  # Evaluation interval
    baseEjectionTime: 30s          # Eject endpoint for 30 seconds
    maxEjectionPercent: 50         # Eject at most 50% of endpoints
```

**In Kong**: Use the `circuit-breaker` plugin or proxy-cache with health checks.

**In application code**: Netflix Hystrix (deprecated), resilience4j (JVM), Polly (.NET), go-resiliency.

---

**Q: How do you implement zero-downtime deployments with an API gateway or reverse proxy?**

**Blue-Green deployment**:
- Maintain two identical production environments: Blue (current) and Green (new version).
- Deploy the new version to Green.
- Run smoke tests on Green.
- Flip the API gateway/LB to route 100% traffic to Green.
- Blue stays running as an instant rollback target.
- After confidence, decommission Blue or repurpose it as the next deploy target.

**Canary deployment**:
- Route a small fraction (1%, 5%) of traffic to the new version.
- Monitor error rates, latency, and business metrics for the canary.
- If metrics are healthy, gradually increase traffic (10% → 50% → 100%).
- If anomalies detected, route 100% back to the old version instantly.

**Kong weighted routing**:
```yaml
upstreams:
  - name: api-upstream
    targets:
      - target: app-v1:8080
        weight: 90
      - target: app-v2:8080
        weight: 10
```

**Nginx upstream weights**: Similar — adjust `weight` parameter per server in upstream block.

**Connection draining**: When removing an old backend, stop sending new requests but allow existing connections to complete. Configure `proxy_read_timeout` and use health checks to detect when the old backend has drained.

**Graceful shutdown**: Application should handle `SIGTERM` by stopping accepting new requests, completing in-flight requests (up to a timeout), then exiting. Kubernetes sends SIGTERM and waits `terminationGracePeriodSeconds` (default 30s) before SIGKILL.
