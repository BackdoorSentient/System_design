# Back-of-Envelope Estimation

**Q: Why do system design interviews require estimation?**
A: Estimation determines the scale of the problem, which determines the architecture. 100 QPS
needs a single server. 1,000,000 QPS needs a distributed fleet, CDN, caching, and database
sharding. Without knowing the numbers, you can't make meaningful design decisions. Interviewers
want to see that you can translate business requirements into technical constraints.

**Q: What numbers should every engineer have memorized?**

Time:
- Seconds in a minute: 60
- Seconds in an hour: 3,600
- Seconds in a day: 86,400 → round to 100,000
- Seconds in a month: ~2.5 million
- Seconds in a year: ~30 million

Storage units:
- 1 KB = 10³ bytes (kilobyte)
- 1 MB = 10⁶ bytes (megabyte)
- 1 GB = 10⁹ bytes (gigabyte)
- 1 TB = 10¹² bytes (terabyte)
- 1 PB = 10¹⁵ bytes (petabyte)

Typical data sizes:
- ASCII character: 1 byte
- Unicode character: 2–4 bytes
- Integer (int32): 4 bytes
- Long (int64) / double: 8 bytes
- UUID: 16 bytes
- IPv4 address: 4 bytes
- IPv6 address: 16 bytes
- Thumbnail image: ~10–50 KB
- Web page (HTML): ~50–100 KB
- High-res photo: ~1–5 MB
- 1 minute of MP3 audio: ~1 MB
- 1 minute of 1080p video: ~100–500 MB

Network speeds:
- Average mobile connection: ~10–50 Mbps
- Average home broadband: ~100 Mbps
- Data center network link: 1–100 Gbps
- SSD read speed: ~500 MB/s (SATA) to ~7 GB/s (NVMe)
- HDD read speed: ~100–150 MB/s
- L1 cache access: ~1 ns
- Main memory (RAM) access: ~100 ns
- SSD random read: ~100 µs
- HDD seek: ~10 ms
- Network round trip (same data center): ~0.5 ms
- Network round trip (cross-continent): ~150–200 ms

**Q: How do you calculate QPS from user numbers?**
A: Formula: QPS = (DAU × actions per user per day) / 86,400

Always calculate:
- Average QPS: baseline load during normal operation.
- Peak QPS: highest load (flash sales, breaking news, morning rush). Typically 3–5× average.
- Design for peak. Infrastructure that handles average will collapse at peak.

Example — Design YouTube:
- 2 billion logged-in users per month, ~500M daily active users
- Average user watches 5 videos/day, each video = 1 view event + 5 recommendation requests
- Requests per user per day: 6
- Total requests per day: 500M × 6 = 3 billion
- Average QPS: 3B / 100,000 = 30,000 QPS
- Peak QPS (5× average): 150,000 QPS

This tells you: you need a fleet of hundreds of servers, global load balancing, and heavy
caching — not a single machine.

**Q: How do you estimate storage requirements?**
A: Formula: Storage = (items written per day) × (average item size) × (retention period in days) × (replication factor)

Example — Design a logging system:
- 1,000 microservices, each emitting 100 log lines per second
- Total: 100,000 log lines/second = 8.64 billion lines/day
- Average log line: 200 bytes
- Raw storage per day: 8.64B × 200B = ~1.7 TB/day
- Retention: 30 days → 51 TB
- Replication factor: 3 → 153 TB total storage needed
- Compression ratio (~10:1 for logs): ~15 TB actual

This tells you: you need a distributed storage system, not a single server.

**Q: How do you estimate bandwidth requirements?**
A: Formula: Bandwidth = QPS × average response size

Example — API serving JSON:
- 50,000 QPS (write: 5,000, read: 45,000)
- Average read response: 2 KB JSON payload
- Average write request: 500 bytes
- Read bandwidth: 45,000 × 2KB = 90 MB/s = 720 Mbps outbound
- Write bandwidth: 5,000 × 500B = 2.5 MB/s = 20 Mbps inbound
- Total outbound: ~720 Mbps → need multiple 1Gbps or one 10Gbps link, plus CDN offloading

**Q: How do you estimate cache requirements?**
A: Use the Pareto principle: 20% of content generates 80% of traffic.
Cache that hot 20% and serve most traffic from memory.

Formula: Cache size = total working dataset size × 0.2

Example — Twitter timeline cache:
- 500M DAU, each with a home timeline of ~200 tweet IDs
- Timeline cache entry: 200 × 8 bytes (tweet ID) = 1.6 KB per user
- Cache for 10% of most active users (50M users): 50M × 1.6KB = 80 GB
- With 128 GB RAM Redis instance you can cache all active timelines

**Q: How do you estimate the number of servers needed?**
A: Formula: Servers = Peak QPS / (requests a single server can handle)

A single commodity server (8-core, 16GB RAM, modern framework) can typically handle:
- 1,000–10,000 QPS for CPU-bound API endpoints
- 10,000–100,000 QPS for cache hits or static content
- 100–1,000 QPS for DB-heavy endpoints with complex queries

Example — 150,000 peak QPS, each request is a simple DB-backed API:
- Each server handles ~2,000 QPS → 150,000 / 2,000 = 75 servers
- Add 30% headroom for rolling deploys and failures → ~100 servers

**Q: Walk through a complete estimation for designing WhatsApp.**
A: Assumptions:
- 2 billion users, 500M DAU
- Each user sends 40 messages/day on average
- 20% of messages contain media (image or video)
- Messages are stored for 30 days on servers; media stored for 1 year

QPS (message sends):
- 500M DAU × 40 messages/day = 20B messages/day
- Write QPS: 20B / 100,000 = 200,000 QPS (writes)
- Read QPS: Assume each message is read by 2 people on average → 400,000 QPS (reads)
- Peak (3×): 600,000 write QPS, 1.2M read QPS

Storage (messages):
- Text message: ~1 KB (with metadata, sender, timestamp, group info)
- 20B messages/day × 1 KB = 20 TB/day text
- Media: 20% × 20B = 4B media messages/day
  - Average media size: 100 KB (mix of images and short videos)
  - 4B × 100 KB = 400 TB/day media
- Total per day: ~420 TB/day
- Text retention 30 days: 20 TB × 30 = 600 TB
- Media retention 1 year: 400 TB × 365 = ~146 PB
- With 3× replication: ~440 PB total

Bandwidth:
- Outbound (reads): 400,000 QPS × 1 KB average = 400 MB/s = ~3.2 Gbps text
- Media delivery: 4B media/day delivered twice = 8B / 86,400 = ~92,000 media QPS
  → 92,000 × 100 KB = 9.2 GB/s = ~73 Gbps (needs heavy CDN offloading)

Servers:
- Message processing: 600,000 peak write QPS / 5,000 QPS per server = 120 servers
- With replication and headroom: ~400 servers for message processing
- Media: Use object storage (S3-equivalent) + CDN edge caching

These numbers explain why WhatsApp uses: Erlang (handles millions of concurrent connections
per server), custom distributed databases, massive object storage, and global CDN infrastructure.

**Q: What are common mistakes in back-of-envelope estimation?**
A:
- Forgetting replication: Always multiply storage by 3 for 3-replica durability.
- Using average instead of peak: Systems fail at peak, not average. Always compute peak.
- Ignoring write amplification: SSDs and databases write more than the user data
  (indexes, WAL, compaction) — actual disk writes can be 5–10× the logical write.
- Forgetting metadata: A 100 KB image also has a row in the metadata DB (URL, owner,
  timestamp, tags) — that's another 500 bytes × billions of images.
- Confusing MB/s and Mbps: 1 Gbps network link = 125 MB/s throughput.
- Not checking reasonableness: If your estimate says you need 1 million servers, double-check.
  If it says one server handles everything, also double-check.

**Q: What is the power of 2 table and why is it useful?**
A:
| Power | Exact value | Approx | Name |
|-------|-------------|--------|------|
| 2^10  | 1,024       | ~1K    | Kilobyte |
| 2^20  | 1,048,576   | ~1M    | Megabyte |
| 2^30  | 1.07×10⁹   | ~1B    | Gigabyte |
| 2^32  | 4.29×10⁹   | ~4B    | Max unsigned 32-bit integer |
| 2^40  | 1.1×10¹²   | ~1T    | Terabyte |

Useful for: estimating if an ID space is large enough (32-bit IDs cap at 4 billion — Twitter
moved to 64-bit Snowflake IDs), reasoning about hash table sizes, bloom filter sizing.