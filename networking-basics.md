# Networking Basics

**Q: What is the OSI model and why does it matter?**
A: The OSI (Open Systems Interconnection) model is a conceptual framework dividing network
communication into 7 layers. Each layer handles a specific responsibility and communicates
with layers directly above and below it.

Layer 1 — Physical: Actual bits on the wire/fiber/radio. Cables, switches, NICs.
Layer 2 — Data Link: Node-to-node transfer, MAC addresses, Ethernet frames. ARP lives here.
Layer 3 — Network: IP addressing and routing. Packets, routers. IP protocol.
Layer 4 — Transport: End-to-end communication. TCP (reliable), UDP (unreliable). Ports.
Layer 5 — Session: Managing connections and sessions.
Layer 6 — Presentation: Encoding, encryption, compression. TLS lives here conceptually.
Layer 7 — Application: HTTP, DNS, SMTP, FTP — protocols applications use directly.

Why it matters: When debugging network issues, the OSI model tells you where to look.
Packet not arriving? Layer 3 (routing). Wrong data? Layer 6 (encoding). Service unavailable?
Layer 7 (application config). Load balancers operate at L4 (TCP) or L7 (HTTP).

**Q: How does TCP work, and what is the 3-way handshake?**
A: TCP (Transmission Control Protocol) provides reliable, ordered, error-checked delivery.

The 3-way handshake establishes a connection before any data is sent:
1. SYN: Client sends a SYN packet with a random sequence number X. "I want to connect."
2. SYN-ACK: Server responds with SYN-ACK, acknowledging X+1 and sending its own sequence
   number Y. "Acknowledged, here's my sequence number."
3. ACK: Client acknowledges Y+1. "Got it. Connection established."

After this, both sides have agreed on sequence numbers and can exchange data reliably.

TCP also handles:
- Acknowledgements: Receiver confirms each packet; sender retransmits if no ACK arrives.
- Flow control: Receiver advertises a window size — how much data it can buffer — to
  prevent sender from overwhelming it.
- Congestion control: TCP detects network congestion (via dropped packets or ECN) and
  slows down transmission to avoid making congestion worse. Algorithms: CUBIC, BBR, Reno.
- Connection teardown: 4-way FIN/FIN-ACK handshake to close cleanly.

TCP connection has overhead: the handshake adds ~1 round trip time before data flows.
TLS adds another 1-2 RTTs on top. This is why HTTP/2 and QUIC (HTTP/3) use persistent
connections and multiplexing to amortize this cost.

**Q: How does UDP work and when is it the better choice?**
A: UDP (User Datagram Protocol) is connectionless — it sends packets (datagrams) with no
handshake, no acknowledgement, no ordering guarantee, no retransmission. It's fire-and-forget.

UDP header is only 8 bytes vs TCP's 20 bytes minimum. No connection state is maintained.

When UDP wins:
- Live video/audio streaming: A dropped frame is better than a delayed one. Retransmission
  would arrive too late to be useful.
- Online gaming: Position updates are sent many times per second. A lost packet is irrelevant
  because the next one arrives 16ms later with a fresher position.
- DNS: Single request-response, no need for connection overhead. If the response is lost,
  the client retries.
- IoT sensor data: Sending thousands of readings per second — losing a few is acceptable.
- VoIP: Same as live audio — latency matters more than reliability.

Custom reliability: Applications like gaming engines sometimes build custom reliability on
top of UDP — retransmitting only critical packets (like "player fired") but not position
updates. This gives them more control than TCP's one-size-fits-all reliability.

**Q: What is HTTP and what are the key differences between HTTP/1.1, HTTP/2, and HTTP/3?**
A: HTTP (HyperText Transfer Protocol) is a request-response application protocol. A client
sends a request (verb + URL + headers + body), a server sends a response (status + headers + body).

HTTP/1.1 (1997): Persistent connections (keep-alive) reuse TCP connections, but requests
are still serial on each connection. To parallelize, browsers open 6 connections per domain.
Head-of-line blocking: one slow response blocks all following responses on that connection.

HTTP/2 (2015): Single TCP connection per domain with multiplexing — multiple requests and
responses interleave on the same connection simultaneously. No head-of-line blocking at
the HTTP layer. Also adds header compression (HPACK) and server push (server sends resources
the client hasn't asked for yet). Widely adopted, major performance improvement.

HTTP/3 (2022): Replaces TCP with QUIC (built on UDP). QUIC has built-in TLS 1.3, 0-RTT
connection establishment (no separate TCP + TLS handshakes), and true stream multiplexing
where packet loss in one stream doesn't block others. Solves TCP head-of-line blocking at
the transport layer. Especially beneficial on lossy mobile networks.

**Q: What is TLS and how does it work?**
A: TLS (Transport Layer Security) provides encryption, authentication, and integrity for
network connections. HTTPS = HTTP over TLS.

TLS 1.3 handshake (simplified):
1. Client Hello: Client sends supported cipher suites and a random value.
2. Server Hello: Server chooses cipher suite, sends its certificate (containing public key)
   and its random value.
3. Client verifies the server certificate against trusted Certificate Authorities (CAs).
4. Key exchange: Both sides use ECDHE (Elliptic Curve Diffie-Hellman) to derive a shared
   session key — without ever transmitting the key itself.
5. Both sides now encrypt all communication with the session key (symmetric encryption,
   e.g. AES-256-GCM). Only 1 RTT for TLS 1.3, vs 2 RTT for TLS 1.2.

Certificate Authorities: CAs (DigiCert, Let's Encrypt, Comodo) digitally sign certificates
to vouch for a server's identity. Your browser/OS ships with a list of trusted root CAs.
If a certificate is signed by a trusted CA, the browser accepts it without warning.

**Q: What is DNS and how does a full resolution work end to end?**
A: DNS (Domain Name System) translates human-readable hostnames into IP addresses. It's a
distributed, hierarchical, cached database replicated across millions of servers worldwide.

Full resolution for `api.github.com`:

1. Browser cache: Browser checks if it has a cached IP for `api.github.com`. If yes, done.
2. OS cache: OS checks /etc/hosts and its DNS cache. If yes, done.
3. Recursive resolver: OS asks the configured recursive resolver (usually your ISP's or
   8.8.8.8). The resolver checks its cache. If yes, done.
4. Root nameservers: Resolver asks a root nameserver (13 root server clusters worldwide)
   "who handles .com?" Root returns the .com TLD nameserver addresses.
5. TLD nameserver: Resolver asks the .com TLD nameserver "who handles github.com?" TLD
   returns GitHub's authoritative nameserver.
6. Authoritative nameserver: Resolver asks GitHub's authoritative nameserver for
   `api.github.com`. Returns the A record: 140.82.x.x
7. Caching: Resolver caches the result for the TTL (time-to-live) specified in the record.
   Browser caches it too.
8. Connection: Browser connects to 140.82.x.x on port 443.

DNS record types:
- A: Domain → IPv4 address
- AAAA: Domain → IPv6 address
- CNAME: Alias → another domain (e.g. www.github.com → github.com)
- MX: Mail server for a domain
- TXT: Arbitrary text (used for SPF, DKIM, domain verification)
- NS: Authoritative nameservers for a domain

**Q: What is a CDN and how does it work technically?**
A: A CDN (Content Delivery Network) is a globally distributed network of servers (points of
presence, or PoPs) that cache content close to end users.

How a CDN serves a request:
1. DNS is configured so `cdn.example.com` points to a CDN nameserver.
2. CDN's DNS returns the IP of the PoP geographically closest to the user (Anycast routing
   or GeoDNS).
3. User's request hits the nearby PoP (e.g. Mumbai for an Indian user).
4. If the PoP has the content cached (cache hit): it responds directly. No trip to origin.
5. If not cached (cache miss): PoP fetches from the origin server, caches the response
   per Cache-Control headers, and serves the user.

Cache-Control headers control CDN behavior:
- `max-age=86400`: Cache for 24 hours.
- `s-maxage=3600`: CDN-specific max age (overrides max-age for CDNs).
- `no-cache`: Must revalidate with origin before serving.
- `Cache-Control: private`: Don't cache (for user-specific responses).

CDN benefits:
- Latency: Mumbai to Mumbai = ~5ms vs Mumbai to US East = ~200ms.
- Bandwidth offload: 90%+ of traffic served from edge, massively reducing origin load.
- DDoS protection: Attack traffic is absorbed by the distributed CDN, not your origin.
- Availability: If your origin is down, CDN can serve stale content (with `stale-if-error`).

**Q: What is a reverse proxy and how does it differ from a load balancer?**
A: A reverse proxy sits between clients and backend servers, forwarding client requests to
backends and returning responses. From the client's perspective, the reverse proxy IS the
server — the client never knows backend servers exist.

Reverse proxy functions: SSL termination, request routing, compression, caching,
authentication, rate limiting, logging.

Load balancer: Specifically distributes requests across multiple backend instances of the
same service. Often a feature of a reverse proxy.

Nginx and HAProxy are common reverse proxies used as load balancers. AWS ALB is a managed
Layer 7 reverse proxy + load balancer. API gateways (Kong, AWS API Gateway) are specialized
reverse proxies for microservices.

**Q: What is a WebSocket and when do you use it?**
A: HTTP is request-response — the client always initiates, the server responds, and the
connection closes (or is reused for the next request). The server cannot push data to the
client without a request.

WebSocket provides a persistent, full-duplex channel over a single TCP connection. Either
side can send messages at any time without a prior request.

Upgrade handshake: Starts as an HTTP request with `Upgrade: websocket` header. If the server
supports it, it responds with 101 Switching Protocols and the connection is upgraded.

Use cases: Real-time chat (Slack, Discord), live dashboards (stock tickers, sports scores),
collaborative editing (Google Docs), multiplayer games, push notifications.

Alternatives:
- Server-Sent Events (SSE): Server pushes to client only, over HTTP. Simpler than WebSocket
  for one-directional push (live feeds, notifications).
- Long polling: Client holds a request open; server responds when data is available. Works
  everywhere but inefficient — each response requires a new request.