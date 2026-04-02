# 23. Service Mesh — Sidecar Pattern & Istio

## What is a Service Mesh?

A service mesh is a dedicated infrastructure layer that handles **service-to-service communication** (east-west traffic) in a microservices architecture. It provides:
- Mutual TLS (mTLS) between services
- Load balancing and traffic routing
- Circuit breaking and retries
- Observability (metrics, traces, logs for every call)
- Access policies (which service can call which)

Without a service mesh, all this logic must be reimplemented in every service, in every language.

---

## Q1: What is the sidecar pattern?

In the sidecar pattern, a proxy container is deployed alongside each application container (in the same pod in Kubernetes). All traffic in and out of the application goes through the sidecar proxy — the application doesn't know it exists.

```
┌─────────────────────────────────┐
│            Kubernetes Pod        │
│  ┌──────────────┐  ┌──────────┐ │
│  │  App         │  │ Sidecar  │ │
│  │  Container   │◄─►│ Proxy   │ │
│  │  :8080       │  │ (Envoy)  │ │
│  └──────────────┘  └──────────┘ │
│                        │         │
└────────────────────────┼─────────┘
                         │
              (all external traffic)
```

**How it intercepts traffic:** Istio uses iptables rules to redirect all inbound/outbound pod traffic to the Envoy proxy (port 15001/15006) transparently.

**Benefits of sidecar:**
- Language/framework agnostic — works for Python, Go, Java, etc.
- No application code changes
- Consistent behavior across all services
- Upgradeable independently of the application

---

## Q2: What is Istio?

Istio is the most widely used service mesh. It has two planes:

### Control Plane (istiod)

- **Pilot:** Converts high-level routing rules into Envoy-compatible configuration and pushes to sidecars
- **Citadel:** Issues and rotates TLS certificates for mTLS
- **Galley:** Validates and distributes configuration

```
istiod (control plane)
    │
    ├── cert management ──► each sidecar (mTLS certificates)
    ├── service discovery ──► each sidecar (cluster topology)
    └── routing config ──► each sidecar (VirtualService, DestinationRule)
```

### Data Plane (Envoy sidecars)

Every pod has an Envoy proxy that:
- Terminates mTLS
- Applies routing rules (canary, A/B testing, traffic shifting)
- Collects metrics and traces
- Enforces circuit breakers and retries

---

## Q3: What traffic management does Istio enable?

### Canary Deployments / Traffic Splitting

```yaml
# Send 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  http:
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10
```

No application changes needed — Envoy enforces the split at the sidecar level.

### Retries and Timeouts

```yaml
http:
- retries:
    attempts: 3
    perTryTimeout: 2s
    retryOn: 5xx,reset,connect-failure
  timeout: 10s
```

### Circuit Breaking

```yaml
trafficPolicy:
  outlierDetection:
    consecutiveErrors: 5
    interval: 1m
    baseEjectionTime: 30s
    maxEjectionPercent: 50
```

If a pod gets 5 consecutive errors in 1 minute, Envoy ejects it for 30s.

---

## Q4: How does mTLS work in a service mesh?

**mTLS (Mutual TLS):** Both the client and server authenticate each other with certificates.

Without mTLS:
- Service A calls Service B over plain HTTP inside the cluster
- Any compromised pod can impersonate Service B
- Traffic is unencrypted (visible to anyone on the network)

With Istio mTLS:
1. Citadel issues each service a X.509 certificate (SPIFFE identity)
2. Service A's Envoy presents its cert when calling Service B's Envoy
3. Service B's Envoy verifies Service A's cert against trusted CA
4. Only authorized identities can connect

```
Service A (Envoy) ──mTLS──► Service B (Envoy)
    cert: spiffe://cluster.local/ns/default/sa/payment-sa
```

**Peer authentication policy:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # reject non-mTLS traffic
```

---

## Q5: Observability with Istio

Every Envoy proxy automatically emits:

**Metrics (to Prometheus):**
- `istio_requests_total` — request count by source, destination, response code
- `istio_request_duration_milliseconds` — latency histogram
- `istio_request_bytes` — payload sizes

**Traces (to Jaeger/Zipkin):**
- Distributed trace for every request, propagated via B3 headers
- Visualize end-to-end call chains across services

**Logs:**
- Access logs for every request at each hop

This is a huge operational win — you get service-level metrics and traces without touching application code.

---

## Q6: Service Mesh Trade-offs

| Pro | Con |
|-----|-----|
| Uniform observability (no code change) | Added latency per hop (~1–5ms) |
| mTLS encryption everywhere | Resource overhead (Envoy sidecar per pod) |
| Traffic management without deploys | Operational complexity (learning curve) |
| Fine-grained access policies | Debugging the mesh itself is hard |
| Consistent retry/timeout behavior | sidecar injection can be a footgun |

**Performance impact:** Envoy adds ~1–5ms of latency per hop and ~50MB RAM per sidecar. For high-throughput internal services, this can matter.

---

## Q7: Alternatives to Istio

| Option | Description |
|--------|-------------|
| **Linkerd** | Simpler, lighter mesh written in Rust; lower overhead |
| **Consul Connect** | Service mesh via Consul; good for hybrid cloud |
| **AWS App Mesh** | Managed mesh on AWS using Envoy |
| **Cilium** | eBPF-based, no sidecar; better performance |
| **No mesh** | Use gRPC with interceptors, or library-based (Hystrix) for smaller systems |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Envoy sidecar memory overhead | ~50MB per pod |
| mTLS cert rotation interval (Istio) | 24h (default) |
| Envoy added latency per hop | ~1–5ms |
| Istio control plane memory | ~1GB for 100-node cluster |

---

## Interview Q&A

**Q: Why use a service mesh instead of handling retries and circuit breakers in the application?**
A: Application-level resilience libraries (Hystrix, Resilience4j) require implementation in every service, in every language, and must be kept in sync. A service mesh moves this to the infrastructure layer — one configuration file gives consistent retry/timeout/circuit-breaker behavior across all services, updated without code deploys. It also adds observability (distributed traces) for free.

**Q: When would you NOT use a service mesh?**
A: Small teams with 5–10 microservices often don't need the operational overhead. The learning curve for Istio is steep. If you don't have dedicated platform engineers, managing Istio can be more pain than it's worth. Linkerd is simpler. For a monolith migrating to microservices, defer the mesh decision until you actually feel the pain it solves.

**Q: How does a sidecar proxy avoid adding round-trip latency to each service call?**
A: The sidecar runs in the same pod (same network namespace) as the application. The connection from app to sidecar is a localhost loopback — effectively zero network latency. The ~1–5ms overhead is from TLS handshake, routing decisions, and telemetry collection, not from network distance.