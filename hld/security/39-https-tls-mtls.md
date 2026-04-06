# 39. HTTPS, TLS & mTLS

## What is TLS?

**TLS (Transport Layer Security)** is the cryptographic protocol that secures communication over a network. HTTPS = HTTP over TLS. It provides:

- **Confidentiality** — data is encrypted; eavesdroppers can't read it
- **Integrity** — data can't be tampered with in transit (MAC/AEAD)
- **Authentication** — client verifies it's talking to the real server (certificates)

TLS 1.3 is the current standard (2018). TLS 1.0 and 1.1 are deprecated. TLS 1.2 still widely used but being phased out.

---

## Q1: How does the TLS 1.3 Handshake work?

TLS 1.3 reduced the handshake from 2 round trips (TLS 1.2) to **1 round trip**.

```
Client                              Server
  │                                    │
  │──── ClientHello ──────────────────►│
  │     (TLS version, cipher suites,   │
  │      key_share: Diffie-Hellman     │
  │      public key)                   │
  │                                    │
  │◄─── ServerHello ───────────────────│
  │     (chosen cipher, DH public key) │
  │◄─── Certificate ───────────────────│
  │◄─── CertificateVerify ─────────────│
  │◄─── Finished (encrypted) ──────────│
  │                                    │
  │     [Both sides derive session keys│
  │      from DH exchange]             │
  │                                    │
  │──── Finished (encrypted) ─────────►│
  │                                    │
  │══════ Encrypted Application Data ══│
```

**Key steps:**
1. Client sends supported ciphers + DH public key
2. Server picks cipher, sends its DH public key + certificate
3. Both compute the same session key (ECDHE — Ephemeral Diffie-Hellman)
4. Session key is never transmitted — derived independently by both sides

**Why Ephemeral DH?** Forward secrecy — even if the server's private key is later compromised, past sessions can't be decrypted because each session used a unique ephemeral key.

### TLS 1.2 vs TLS 1.3

| | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Handshake round trips | 2 | 1 |
| 0-RTT (resumed sessions) | No | Yes (with caveats) |
| Forward secrecy | Optional | Mandatory |
| Cipher flexibility | Many (some weak) | Only strong ciphers |
| RSA key exchange | Supported | Removed |
| Latency overhead | ~100ms | ~50ms |

---

## Q2: What are TLS certificates and how do they work?

A certificate is a **digitally signed document** that binds a public key to an identity (domain name).

### Certificate Chain

```
Root CA (trusted, pre-installed in OS/browser)
    └── Intermediate CA (signed by Root CA)
            └── Server Certificate (signed by Intermediate CA)
                 domain: api.example.com
                 public key: [RSA/EC public key]
                 valid: 2024-01-01 to 2025-01-01
```

**Verification process:**
1. Server sends its certificate + intermediate CA cert
2. Client verifies intermediate CA is signed by a trusted Root CA (in browser/OS trust store)
3. Client verifies server cert is signed by intermediate CA
4. Client verifies cert's domain matches the requested domain (CN or SAN)
5. Client verifies cert is not expired and not revoked (OCSP/CRL)

### Certificate Types

| Type | Validates | Use Case |
|------|-----------|---------|
| **DV (Domain Validated)** | Domain ownership only | Most websites, APIs |
| **OV (Organization Validated)** | Domain + organization | Corporate sites |
| **EV (Extended Validation)** | Domain + org + legal | Banks (green bar, largely deprecated) |
| **Wildcard** | `*.example.com` | All subdomains |
| **SAN (Subject Alt Name)** | Multiple domains in one cert | Multi-domain services |

### Let's Encrypt

Free, automated DV certificate authority. Certificates valid for **90 days** (forces automation). `certbot` or ACME protocol handles automatic renewal.

---

## Q3: What is Certificate Pinning?

**Certificate pinning** means the client hardcodes the expected certificate (or its public key hash) and rejects any valid certificate that doesn't match — even if it's properly signed by a trusted CA.

**Why?** Defends against CA compromise. If a rogue CA issues a fraudulent cert for `api.stripe.com`, a normal TLS handshake would accept it (valid chain). Pinning would reject it.

```
// Mobile app pins Stripe's certificate public key hash
const PINNED_KEY = "sha256/AbCdEf...";

if (serverCert.publicKeyHash != PINNED_KEY) {
    throw new SecurityException("Certificate pinning failure");
}
```

**Risks of pinning:**
- If you rotate your cert and forget to update the pin → app breaks for all users
- Hard to update in native mobile apps (requires app store update)
- Over-pinning (pinning leaf cert) is more fragile than pinning intermediate/root

**Best practice:** Pin to the intermediate or root CA, not the leaf cert. Use backup pins. Have a "pin update" mechanism.

**Used by:** Stripe, Twitter, high-security mobile apps

---

## Q4: What is mTLS (Mutual TLS)?

Standard TLS authenticates the **server** to the client. **mTLS** adds authentication in both directions — the server also verifies the client's certificate.

```
Standard TLS:
Client ──── verifies server cert ────► Server
(client remains anonymous to server)

mTLS:
Client ──── verifies server cert ────► Server
Client ◄─── verifies client cert ───── Server
(both sides authenticated by certificate)
```

### mTLS Handshake

```
Client                              Server
  │──── ClientHello ──────────────────►│
  │◄─── ServerHello + Certificate ─────│
  │◄─── CertificateRequest ────────────│  ← server requests client cert
  │──── ClientCertificate ────────────►│
  │──── CertificateVerify ────────────►│
  │──── Finished ──────────────────────►│
  │◄─── Finished ───────────────────────│
  │══════ Encrypted Application Data ══│
```

### Use Cases for mTLS

**1. Service-to-service authentication (microservices)**
Every service has a certificate with its identity (SPIFFE format). Services only accept requests from other services with valid certs.
```
payment-service cert: spiffe://cluster.local/ns/prod/sa/payment
order-service cert: spiffe://cluster.local/ns/prod/sa/order

order-service → payment-service: "I'm order-service (cert)"
payment-service verifies: "Yes, that's a valid cert from my cluster" → allowed
random pod: no valid cert → rejected
```

**2. Zero-trust network**
No implicit trust even inside the internal network. Every connection authenticated via mTLS.

**3. API client authentication**
Enterprise APIs (financial, healthcare) issue certificates to clients instead of API keys. Harder to steal than a static key.

### mTLS with Istio

Istio automatically provisions and rotates mTLS certificates for every pod using SPIFFE/SPIRE:

```yaml
# Enforce strict mTLS for the entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # reject non-mTLS traffic
```

---

## Q5: What is HSTS (HTTP Strict Transport Security)?

HSTS tells browsers: **"Only ever connect to this domain over HTTPS. Never HTTP."**

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- `max-age=31536000` — remember this for 1 year
- `includeSubDomains` — apply to all subdomains
- `preload` — submit to browser preload lists (hardcoded into Chrome/Firefox source)

**HSTS prevents:**
- SSL stripping attacks (downgrade from HTTPS to HTTP)
- Mixed content
- Accidental HTTP bookmarks

**HSTS Preload:** Once on the preload list, browsers will NEVER attempt HTTP to your domain — even on first visit, before receiving the HSTS header.

---

## Q6: Common TLS Vulnerabilities

| Vulnerability | Description | Fix |
|---------------|-------------|-----|
| **BEAST** | CBC mode attack in TLS 1.0 | Use TLS 1.2+ |
| **POODLE** | SSLv3/TLS 1.0 padding oracle | Disable SSLv3, TLS 1.0 |
| **HEARTBLEED** | OpenSSL memory leak bug | Patch OpenSSL |
| **BREACH** | HTTP compression + TLS = secret extraction | Disable HTTP compression for sensitive data |
| **SSL Stripping** | Downgrade HTTPS to HTTP | HSTS + preload |
| **Expired cert** | Certificate not renewed | Auto-renewal (Let's Encrypt/certbot) |
| **Self-signed cert** | No CA validation | Never in production |
| **Weak ciphers** | RC4, DES, export ciphers | Enforce strong cipher suites |

**TLS configuration best practice:**
```nginx
# nginx TLS hardening
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;
ssl_prefer_server_ciphers off;  # TLS 1.3 ignores this anyway
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
add_header Strict-Transport-Security "max-age=31536000" always;
```

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| TLS 1.3 handshake latency | ~50ms (1 RTT) |
| TLS 1.2 handshake latency | ~100ms (2 RTT) |
| TLS 1.3 0-RTT latency | ~0ms (resumed sessions, replay risk) |
| Let's Encrypt cert validity | 90 days |
| Recommended cert validity (internal) | 1 year max |
| HSTS max-age (Google recommendation) | 2 years (63072000s) |
| Istio cert rotation interval | 24 hours (default) |
| RSA key size minimum (2024) | 2048 bits (prefer 4096) |
| ECDSA key size equivalent | 256 bits ≈ 3072-bit RSA |

---

## Interview Q&A

**Q: What is forward secrecy and why does TLS 1.3 mandate it?**
A: Forward secrecy means that compromising the server's long-term private key today doesn't compromise past sessions. In TLS 1.2 with RSA key exchange, the client encrypts the session key with the server's public key — if the private key is stolen later, an attacker who recorded past traffic can decrypt it. TLS 1.3 mandates Ephemeral Diffie-Hellman (ECDHE) — the session key is derived from a fresh ephemeral key per session. Even with the server's private key, past sessions can't be decrypted.

**Q: How does mTLS differ from API key authentication?**
A: API keys are static shared secrets — if stolen, an attacker has indefinite access until the key is rotated. mTLS uses asymmetric cryptography — the private key never leaves the client, so it can't be stolen from transit. mTLS certificates have built-in expiry and can be automatically rotated (Istio does this every 24h). API keys require manual rotation. For service-to-service communication, mTLS is strictly stronger. API keys are still useful for external developers (simpler to use, no certificate infrastructure needed).

**Q: What is TLS termination and where should you do it?**
A: TLS termination is where encrypted traffic is decrypted. Options: (1) At the load balancer/CDN — decrypted traffic travels unencrypted inside your network (simpler, faster, but internal traffic is plaintext). (2) End-to-end (passthrough) — TLS all the way to the backend service (stronger but more CPU overhead, harder to inspect traffic). (3) Re-encryption — terminate at LB, re-encrypt to backend. For internal microservices, use mTLS for east-west traffic. For north-south (external), terminate at the edge (CDN/load balancer) where you can do DDoS protection and WAF inspection.