# Case Study: WhatsApp

---

**Q: What are the requirements for a WhatsApp-like messaging system?**

**Functional requirements:**
- 1:1 messaging: send and receive text messages between two users
- Group messaging: up to 256 members per group
- Media: send images, video, audio, documents
- Message delivery receipts: sent ✓, delivered ✓✓, read ✓✓ (blue ticks)
- Online presence: show whether a contact is online or was last seen at a given time
- End-to-end encryption (E2EE): server cannot read message content
- Push notifications when offline

**Non-functional requirements:**
- **Low latency**: Messages should arrive in < 100ms on a good connection
- **High availability**: 99.99% — messaging is real-time; outages are very visible
- **Durability**: Messages must not be lost in transit. If a recipient is offline, deliver when they reconnect
- **Scale**: 2B users, 100B messages/day
- **Ordered delivery**: Messages must arrive in the order they were sent (per conversation)

---

**Q: How do you estimate scale for WhatsApp?**

**Messages:**
- 100B messages/day = 100B / 86400 ≈ **1.16M messages/sec** average.
- Peak (~3× average): ~3.5M messages/sec.

**Active connections:**
- 2B users, ~50% active daily = 1B DAU.
- Assume each active user has the app open and maintains a persistent connection for ~4 hours/day.
- Concurrent connections at any given time: 1B × (4/24) ≈ **167M concurrent WebSocket connections**.

**Storage:**
- Text message: sender_id (8B) + recipient_id (8B) + message_id (8B) + body (200B avg) + timestamp (8B) + status (1B) ≈ 250 bytes.
- 100B × 250 bytes = 25 TB/day. With media (assume 30% of messages have media, avg 200KB each):
  - 100B × 0.3 × 200KB = 6,000 TB/day = 6 PB/day of media. Stored in object storage (S3/GCS).
- Text messages: ~9 PB/year. Retained for a configurable period (WhatsApp stores on device, not server, post-delivery).

**Key insight from estimation:** The hardest problems are (1) maintaining 167M concurrent connections efficiently, and (2) guaranteed delivery to offline users.

---

**Q: How do you maintain persistent connections at massive scale? What protocol should you use?**

Each connected client must maintain a **persistent bidirectional connection** to the server so the server can push messages without the client polling.

**Options:**

| Protocol | Pros | Cons |
|----------|------|------|
| HTTP long polling | Works everywhere, no special infra | High latency (poll interval), wasted connections |
| Server-Sent Events (SSE) | Simple, HTTP/2 compatible | Unidirectional (server → client only) |
| WebSocket | Full-duplex, low overhead after handshake | Stateful — servers must track which client is on which server |
| XMPP (over TCP) | Purpose-built for messaging, federation | Verbose XML, complex |
| Custom TCP (WhatsApp's actual approach) | Maximum efficiency, Erlang's actor model | Complex to implement |

**WhatsApp's real architecture:** Built on **Ejabberd** (an Erlang XMPP server). Erlang was chosen because:
- Erlang's actor model spawns a lightweight process per connection (~2KB overhead vs ~1MB for OS threads).
- Erlang handles millions of concurrent connections on a single server.
- WhatsApp reportedly ran 2M concurrent connections per server node at peak with 50 engineers.

**For interview purposes:** Use **WebSocket** over HTTPS/TLS. Clients maintain a persistent WebSocket to a connection server. Each WebSocket connection is ~50KB of server memory. 167M connections × 50KB = ~8.35 TB RAM total across your connection server fleet. With 256 GB RAM per server, you need ~33,000 connection servers.

---

**Q: How does message delivery work end-to-end? Walk through the architecture for online and offline recipients.**

**Scenario 1: Both users online**

```
Alice (client) → WebSocket → Connection Server A
                                    ↓
                             Message Router / Broker
                                    ↓
                             Connection Server B → WebSocket → Bob (client)
```

1. Alice sends message via WebSocket to her connection server.
2. Connection server writes message to the message store (for durability) and looks up which connection server Bob is on (via the Session Service).
3. Message is forwarded to Bob's connection server.
4. Bob's server delivers the message over Bob's WebSocket.
5. Bob's client sends back an ACK → server marks message as `delivered`.
6. Alice is notified: Bob received the message → double tick.

**Latency:** < 100ms on a good network. The lookup + forward adds ~5–20ms.

**Scenario 2: Bob is offline**

1. Steps 1–2 above. Message is written to the message store.
2. Session Service reports Bob has no active connection.
3. Message stays in the **offline message queue** (keyed by recipient_id).
4. A push notification is sent to Bob's device (APNs for iOS, FCM for Android).
5. When Bob reconnects: his client sends a "fetch pending messages" request.
6. Server delivers all queued messages in order.
7. Bob's client ACKs each message → server marks delivered, notifies Alice.

**The message store must be durable** — if the server crashes between step 2 and Bob reconnecting, the message must not be lost. Use a persistent store (Cassandra) not Redis (which can lose data on crash).

---

**Q: What is the session service and how does it map users to connection servers?**

With tens of thousands of connection servers, the message router must know **which server holds a given user's WebSocket** to forward messages.

**Session Service:** A distributed key-value store mapping `user_id → connection_server_id`.

- Written on connect: `SET session:{user_id} = server_42` with a TTL (e.g., 30s, refreshed by heartbeat).
- Deleted on disconnect: `DEL session:{user_id}`.
- Read on every message send to find the recipient's server.

**Implementation:**
- Redis is ideal: sub-millisecond GET, built-in TTL, easy replication.
- For 2B users with 50% online: 1B entries × ~30 bytes = 30 GB. Fits comfortably in Redis Cluster.

**What if the user has multiple devices?**
- `session:{user_id}` maps to a **list** of connection server IDs (one per active device).
- Message is delivered to all devices simultaneously.

**Heartbeat to keep session alive:**
- Client sends a small ping every ~20s.
- Server resets the TTL on `session:{user_id}`.
- If no heartbeat for 30s, TTL expires → user is treated as offline.

---

**Q: How do you store messages? What is the data model?**

WhatsApp uses a **client-centric storage model**: messages are stored primarily on devices, not on WhatsApp's servers. The server stores a message only until it is delivered to all recipients — then it is deleted.

This simplifies storage requirements dramatically and is consistent with E2EE (the server holds ciphertext it cannot read, only temporarily).

**Server-side message store (Cassandra):**

```
Table: messages
Partition key: (conversation_id)    ← all messages in a conversation co-located
Clustering key: (message_id DESC)   ← time-ordered within partition, newest first

Columns:
  message_id   timeuuid    (globally unique, contains timestamp)
  sender_id    bigint
  ciphertext   blob        (E2EE: server stores encrypted bytes only)
  media_url    text        (nullable; points to encrypted object in S3)
  status       tinyint     (0=pending, 1=delivered, 2=read)
  created_at   timestamp
  expires_at   timestamp   (delete after delivery or 30 days if undelivered)
```

**Why Cassandra:**
- Write-optimised (LSM tree): 1.16M writes/sec is a natural fit.
- Linear horizontal scaling: add nodes as volume grows.
- Time-ordered clustering key gives efficient range scans for "fetch all messages since X."
- Built-in TTL: set `expires_at` and Cassandra auto-deletes without a separate cleanup job.

**Group messages:**
- A group message is sent to one `conversation_id`.
- Server fans out delivery to all N members (one delivery record per member).
- Delivery receipt is per-member: "delivered to 5 of 6 members" requires tracking each.

---

**Q: How do message delivery receipts (ticks) work at the protocol level?**

WhatsApp's three-state delivery system requires tracking acknowledgement from multiple hops.

**States:**
- **Sent (✓):** Server has received and stored the message. Alice's client got an ACK from the server.
- **Delivered (✓✓):** Bob's device has received the message. Bob's client sent an ACK to the server; server forwarded this ACK to Alice.
- **Read (✓✓ blue):** Bob has opened the conversation and seen the message. Bob's client sends a "read receipt" event.

**Implementation:**

```
Alice sends message → Server stores it → Server ACKs to Alice → Alice shows ✓

Server delivers to Bob → Bob's client ACKs → Server updates status to "delivered"
                      → Server pushes delivery receipt to Alice's WebSocket → Alice shows ✓✓

Bob opens the chat → Bob's client sends read receipt event to server
                   → Server pushes to Alice → Alice shows ✓✓ (blue)
```

**At-least-once delivery:** The server retries delivery until it receives an ACK from Bob's client. Bob's client must deduplicate (message_id is idempotency key).

**Group delivery receipts:**
- "Delivered" shows when delivered to all members.
- "Read" shows when read by all members.
- Requires tracking a separate delivery/read state per (message_id, user_id) pair.

---

**Q: How does end-to-end encryption work in WhatsApp (Signal Protocol)?**

WhatsApp uses the **Signal Protocol**, designed by Open Whisper Systems.

**Key primitives:**
- Each user generates a long-term **Identity Key** pair on device.
- Each user pre-generates ~100 **One-Time Prekeys** (OTKs) and uploads their public halves to the WhatsApp key server.
- The server stores only public keys — it never sees private keys.

**Starting a conversation (X3DH — Extended Triple Diffie-Hellman):**
1. Alice fetches Bob's identity key and one of his OTKs from the server.
2. Alice and Bob perform a key exchange using their identity keys + ephemeral keys + OTK.
3. This generates a shared secret known only to Alice and Bob — **the server never sees this secret**.
4. The OTK is consumed and deleted; a new one is used for each new session.

**Message encryption (Double Ratchet):**
- Every message is encrypted with a different key, derived by advancing a cryptographic ratchet.
- Compromising one message key does not compromise past or future keys (**forward secrecy** and **break-in recovery**).
- Ratchet state is stored only on devices.

**What the server sees:**
- The sender and recipient (metadata — "who talked to whom and when").
- The message ciphertext (opaque encrypted bytes).
- The message size and timestamp.
- The server cannot decrypt the content. This is true E2EE.

**Key verification:** Users can compare "Safety Numbers" (a fingerprint of both parties' identity keys) in person or via another channel to verify no man-in-the-middle has replaced their keys.

---

**Q: How does presence (online/last seen) work at scale?**

**Online presence:**
- When a user connects, publish a `user_online` event to a Presence Service.
- Presence Service writes `{user_id: online, last_seen: now}` to Redis with a 30s TTL.
- Heartbeat from client every 20s resets TTL.
- When a user disconnects (or TTL expires), publish a `user_offline` event; update `last_seen`.

**Subscribing to presence:**
- When Alice opens a chat with Bob, her client subscribes to Bob's presence.
- The Presence Service maintains a pub/sub topic per user.
- Updates fan out to all subscribers (users who have Bob's chat open).

**Scale challenge:** If Bob has 1,000 contacts and all 1,000 are online with his chat open, a single presence update fans out to 1,000 subscribers. At 2B users this is an enormous pub/sub problem.

**Optimisations:**
- Only fans out presence to users who have the chat actively open (not just in the contact list).
- Batch presence updates (don't publish every heartbeat; publish only on status change).
- Deliver presence lazily: when Alice opens a chat, fetch Bob's current status once, then subscribe for changes.

**Privacy:** WhatsApp allows users to hide last seen from all, contacts, or selected users. This is a read-filter applied before returning presence data — not a write-filter.

---

**Q: How do you handle media (images, videos)?**

Sending media directly over the messaging channel would be enormously inefficient (a 10MB video through message routing infrastructure is wasteful).

**Media upload flow:**
```
1. Alice selects a photo
2. Alice's app encrypts the photo with a random key (media key)
3. App uploads ciphertext to WhatsApp's media server (S3-backed) → gets a media URL
4. Alice sends a message containing: { media_url, media_key (encrypted for Bob), media_type }
5. Message is delivered to Bob via normal message path
6. Bob's app downloads ciphertext from media_url
7. Decrypts with the media_key from the message
```

**Why this design:**
- Media doesn't transit the message routing infrastructure — only the URL + encrypted key does.
- The media server stores only ciphertext it cannot decrypt.
- Large files don't congest the WebSocket / messaging path.
- Media is retained on the server for 30 days after sending (in case delivery is delayed). After that, the URL expires.

**CDN for media:**
- Frequently requested media (common stickers, GIFs) is served from CDN edge.
- User-uploaded media is served from regional S3-backed object storage.

---

**Q: What does the full system architecture look like?**

```
Client (iOS/Android)
        ↓ (WebSocket over TLS)
Load Balancer (Layer 4, TCP passthrough)
        ↓
Connection Servers (stateful; 1 process per connection — Erlang/Go)
        ↓                    ↑
  Message Router      Session Service (Redis: user_id → server_id)
        ↓
  Message Store (Cassandra — write-heavy, append-only)
        ↓
  Delivery ACK path  →  Presence Service (Redis pub/sub)
        ↓
  Push Notification Service (APNs / FCM for offline users)

Media path:
  Client → Media Upload Service → S3 (encrypted)
  Client → CDN / S3 (download)

Key server:
  Stores public identity keys + prekeys for E2EE bootstrap
```

**Key decisions summarised:**

| Decision | Choice | Reason |
|----------|--------|--------|
| Connection protocol | WebSocket (or Erlang TCP) | Persistent, bidirectional, low overhead |
| Session tracking | Redis (user → server mapping) | Sub-ms lookup, TTL-based expiry |
| Message storage | Cassandra | Write-heavy, TTL support, append-only |
| Message retention | Until delivery (then deleted) | E2EE model; storage efficiency |
| Media | S3 + CDN, URL in message | Decouples large payloads from routing |
| Encryption | Signal Protocol (E2EE) | Server cannot read content |
| Delivery guarantee | At-least-once with dedup by message_id | No message loss; idempotent on retry |
| Presence | Redis pub/sub with TTL heartbeat | Lightweight, eventually consistent |