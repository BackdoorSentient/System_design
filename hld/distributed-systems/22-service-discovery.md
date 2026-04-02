# 22. Service Discovery

## What is Service Discovery?

In a microservices architecture, services need to find each other's network locations (IP + port) to communicate. Unlike a monolith, microservice instances:
- Have dynamic IPs (containers restart, scale in/out)
- Have multiple instances behind a service name
- Come and go constantly (deployments, autoscaling, failures)

**Service discovery** is the mechanism by which services find and connect to each other dynamically.

---

## Q1: What are the two main discovery patterns?

### 1. Client-Side Discovery

The client (calling service) queries the **service registry** directly, then makes its own load-balancing decision.

```
Service A ──query──► [Service Registry]
          ◄──[IP1, IP2, IP3]──
          ──choose IP2──► Service B (IP2)
```

**Flow:**
1. Service A wants to call Service B
2. Service A asks the registry: "Where are instances of Service B?"
3. Registry returns a list of healthy instances
4. Service A picks one (round-robin, least-connections, etc.) and makes the request

**Pros:**
- Client can implement smart load balancing (weighted, circuit-breaker aware)
- One fewer network hop (no proxy)

**Cons:**
- Every client must implement discovery logic
- Discovery library needed for each language
- Clients must handle stale registry data

**Examples:** Netflix Ribbon + Eureka, Consul client libraries

---

### 2. Server-Side Discovery

The client calls a **load balancer** (or service proxy). The load balancer queries the registry and forwards.

```
Service A ──request──► [Load Balancer / Proxy]
                              │
                        ──query──► [Service Registry]
                        ◄──[IP1, IP2]──
                              │
                        ──forward──► Service B (IP1)
```

**Flow:**
1. Service A sends request to a well-known address (load balancer)
2. Load balancer queries registry for healthy Service B instances
3. LB forwards request to a selected instance

**Pros:**
- Client is simple — no discovery logic
- Works with any language/framework
- Load balancer is the single place for routing logic

**Cons:**
- Load balancer is a potential bottleneck/SPOF (mitigated with HA LB setup)
- Extra network hop

**Examples:** AWS ALB + ECS service discovery, Kubernetes Services, HAProxy + Consul

---

## Q2: What is a service registry and how do services register?

A service registry is a database of service instances and their locations (IP, port, health status).

### Registration Methods

**Self-registration:**
Services register themselves on startup and deregister on shutdown.
```
Service B starts → sends POST /register {service: "B", ip: "10.0.1.5", port: 8080}
Service B stops  → sends DELETE /deregister
```
Problem: If the service crashes without deregistering, the registry has stale entries.
Fix: **TTL-based registration** — services send periodic heartbeats. If no heartbeat within TTL, the registry removes the entry.

**Third-party registration:**
An external agent (e.g., a Kubernetes controller, a consul agent sidecar) monitors services and handles registration/deregistration.
```
k8s controller watches pod lifecycle → registers pod IP in etcd → kube-proxy routes to it
```
Services don't need any discovery library. The platform handles it.

---

## Q3: What is Consul and how does it work?

**Consul** (by HashiCorp) is a service mesh and service discovery tool.

**Components:**
- **Consul agent:** Runs on every node (daemon). Handles local service registration and health checking.
- **Consul server:** 3–5 servers running Raft consensus, storing the service registry.
- **DNS or HTTP API:** Services query Consul for other services.

**Health Checking:**
```yaml
service:
  name: "payment-service"
  port: 8080
  check:
    http: "http://localhost:8080/health"
    interval: "10s"
    timeout: "1s"
    deregisterCriticalServiceAfter: "30s"
```

**DNS-based discovery:**
```
curl payment-service.service.consul → A record → 10.0.1.5
```

**Used by:** Kubernetes (as an alternative), HashiCorp stack, many on-prem setups.

---

## Q4: How does ZooKeeper provide service discovery?

ZooKeeper is a distributed coordination service (not built specifically for service discovery, but often used for it).

**Mechanism:** Services create **ephemeral znodes** (temporary nodes) in ZooKeeper's tree.

```
/services
  /payment-service
    /instance-1  → {ip: "10.0.1.5", port: 8080}   ← ephemeral
    /instance-2  → {ip: "10.0.1.6", port: 8080}   ← ephemeral
```

**Ephemeral znode:** Automatically deleted when the creating session (TCP connection) closes. If service B crashes, its session dies → its znode disappears → clients are notified via ZooKeeper **watchers**.

**Service A watches** `/services/payment-service` → gets notified when instances come/go.

**Pros:** Strong consistency (via Zab consensus), reliable watches, battle-tested
**Cons:** Complex to operate, not purpose-built for discovery, clients must handle watch re-registration

---

## Q5: How does Kubernetes service discovery work?

Kubernetes has built-in service discovery via **Services** and **DNS**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
```

- `kube-dns` creates an A record: `payment-service.default.svc.cluster.local → ClusterIP`
- `ClusterIP` is a virtual IP; `kube-proxy` (iptables/IPVS) routes it to a healthy pod
- When pods scale, `Endpoints` object is updated; kube-proxy reconfigures routing

**Three layers:**
1. **DNS** → resolves service name to ClusterIP
2. **ClusterIP (kube-proxy)** → distributes to pod IPs
3. **Readiness probes** → only route to pods that pass health checks

---

## Q6: Consul vs ZooKeeper vs Kubernetes Service Discovery

| | Consul | ZooKeeper | Kubernetes |
|---|---|---|---|
| Purpose-built for discovery | Yes | No | Yes (within k8s) |
| Health checking built-in | Yes | No (DIY) | Yes (readiness probes) |
| Consistency | CP (Raft) | CP (ZAB) | CP (etcd) |
| DNS support | Yes | No | Yes |
| Service mesh support | Yes (Consul Connect) | No | Via Istio/Linkerd |
| Operational complexity | Medium | High | Low (if on k8s) |

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Consul health check interval | 10s (typical) |
| ZooKeeper session timeout | 30s (typical) |
| Kubernetes readiness probe interval | 10s (default) |
| DNS TTL for service discovery | 5–30s (short, to handle changes) |

---

## Interview Q&A

**Q: What happens if the service registry goes down?**
A: Services that already have the registry data cached can continue operating. New services starting or routes that haven't been cached will fail to resolve. Mitigation: (1) Registry itself is replicated (Consul uses Raft with 3–5 servers, ZooKeeper uses quorum). (2) Clients cache the last-known list of instances and use it during registry unavailability. (3) For Kubernetes, kube-dns is replicated and uses coreDNS with HA replicas.

**Q: Client-side vs server-side discovery — which do you prefer?**
A: For Kubernetes-based systems, server-side (via k8s Services) is the right default — it's built-in, requires no library, and works for any language. For more complex routing needs (circuit breaking, retries, A/B testing), a service mesh (Istio) adds server-side intelligence. Client-side discovery (Netflix Ribbon pattern) made sense in the pre-k8s era but adds complexity without significant benefit today.

**Q: How do you handle service discovery across multiple data centers?**
A: Consul has multi-datacenter support built-in. Services in DC1 can query `service.dc2.consul` to find instances in DC2. ZooKeeper would require federation or a separate cluster per DC. For Kubernetes, tools like Submariner or Istio with multi-cluster federation handle cross-cluster discovery.