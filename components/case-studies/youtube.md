# Case Study: YouTube

---

**Q: What are the requirements for a YouTube-like video platform?**

**Functional requirements:**
- Upload videos (any resolution; platform transcodes to multiple formats)
- Stream videos (adaptive bitrate; works on slow and fast connections)
- Search videos by title, description, tags
- Comments, likes, view counts
- Recommendations / suggested videos
- Subscriptions and a subscription feed

**Non-functional requirements:**
- **Upload durability**: A video once uploaded and processed must never be lost
- **Streaming latency**: Video playback must start in < 2 seconds; buffering should be rare
- **Availability**: Read path (watching) must be highly available; brief upload failures are tolerable
- **Scale**: 500 hours of video uploaded per minute; 1B hours of video watched per day
- **Global reach**: Users in every country; content must be served from geographically close CDN nodes
- **Read-heavy**: The ratio of viewers to uploaders is ~1000:1 or higher

---

**Q: How do you estimate scale for YouTube?**

**Uploads:**
- 500 hours of video uploaded per minute = 500 × 60 = 30,000 minutes of video/minute.
- That's 30,000 / 60 = 500 hours/min, or ≈ **8.3 hours of video per second** being uploaded.

**Storage (uploaded video):**
- 1 minute of 1080p video ≈ 500 MB raw. Compressed H.264: ~150 MB.
- Transcoding to 5 formats (144p, 360p, 720p, 1080p, 4K) per video: ~1.5× the storage of one format.
- 30,000 min/min × 150 MB × 1.5 = **6.75 TB of new video storage per minute**.
- Per year: ~3.5 exabytes of video storage. Stored in distributed object storage (GCS/S3-equivalent).

**Video views:**
- 1B hours watched/day = 1B × 3,600s / 86,400s ≈ **41.7M concurrent viewers** on average.
- Each viewer streams at ~2–8 Mbps (adaptive).
- At avg 4 Mbps: 41.7M × 4 Mbps = **167 Tbps** of video egress bandwidth.
- This is why YouTube is one of the largest consumers of internet bandwidth in the world — and why CDN is non-negotiable.

**Metadata:**
- Video metadata (title, description, tags, uploader, view count): ~5 KB per video.
- 800M total videos × 5 KB = 4 TB. Trivially manageable in a database.

---

**Q: Walk through the video upload and processing pipeline.**

Uploading and making a video watchable involves several stages. The upload experience must be fast; transcoding is asynchronous.

**Step 1: Upload**

```
Client → (resumable chunked upload) → Upload Service → Raw Video Store (object storage)
```

- Client splits video into chunks (e.g., 5 MB each) and uploads sequentially or in parallel.
- **Resumable uploads (Google's resumable upload protocol):** If the upload is interrupted, the client resumes from the last acknowledged chunk rather than restarting. Each chunk gets an offset; the server tracks the byte offset received.
- On completion, the Upload Service writes metadata (title, uploader, raw file path) to the DB and publishes a `video_uploaded` event to Kafka.

**Step 2: Transcoding**

```
Kafka (video_uploaded event)
    ↓
Transcoding Service (distributed, CPU-heavy workers)
    ↓
Produces: video_id/720p.mp4, video_id/360p.mp4, video_id/1080p.mp4, ...
    ↓
Transcoded Video Store (object storage, CDN-backed)
```

- Transcoding is the most CPU-intensive operation. YouTube uses a fleet of dedicated transcoding workers.
- Each resolution is transcoded in parallel by different workers.
- Transcoding 1 hour of 4K video to all formats can take 10–60× real-time on commodity hardware (so 10–60 hours of CPU time).
- For fast turnaround, fan out transcoding across many workers concurrently.
- Output format: HLS (HTTP Live Streaming) or DASH — the video is segmented into 2–10 second `.ts` or `.mp4` fragments, each independently downloadable.

**Step 3: Post-processing**

After transcoding:
- Generate thumbnail candidates (sample frames at regular intervals; ML model selects the most engaging).
- Run content moderation (violence, copyright fingerprinting via Content ID).
- Extract metadata for search indexing.
- Publish `video_ready` event — video becomes visible/searchable.

**Step 4: CDN distribution**

Transcoded segments are pushed to (or lazily pulled into) CDN edge nodes globally. Users stream from the nearest edge node.

---

**Q: What is adaptive bitrate streaming (ABR) and how does it work?**

Users have different bandwidth and devices. Serving a 1080p stream to someone on 3G wastes bandwidth and causes buffering. ABR solves this by switching quality levels mid-stream.

**How HLS (HTTP Live Streaming) works:**

The transcoding pipeline produces a **master playlist** (`.m3u8` file) that lists all available quality levels:

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/index.m3u8
```

Each quality-level playlist lists short video segments:

```
#EXTM3U
#EXT-X-TARGETDURATION:6
seg_001.ts   (6 seconds, 3.75 MB at 5Mbps)
seg_002.ts
seg_003.ts
```

**The ABR algorithm (in the client player):**

1. Client downloads the master playlist and picks a starting quality based on estimated bandwidth.
2. Client downloads segments sequentially, maintaining a buffer (typically 15–30 seconds ahead).
3. After each segment download, the player measures the actual download speed and compares it to the segment bitrate.
4. If download is fast (buffer growing): switch up to a higher quality level on the next segment.
5. If download is slow (buffer shrinking): switch down to a lower quality level.
6. Quality switches happen at segment boundaries — seamless to the viewer.

**Why segments matter for CDN:**
- Each 6-second segment is an independently cacheable HTTP resource.
- CDN caches segments by URL. A 1080p segment of a popular video is cached at every edge node worldwide.
- Most video playback is cache hits at the CDN edge — the origin storage is rarely hit for popular videos.

---

**Q: How do you serve video at global scale? What does the CDN architecture look like?**

YouTube's CDN (Google's own — not a third-party CDN) operates at a scale few systems match.

**Three-tier CDN:**

```
Origin (Google data center)
    ↓ (video first uploaded here)
Regional Cache (one per major region: US-East, EU-West, APAC, etc.)
    ↓ (regional caches pull from origin on first miss)
Edge Cache / PoP (Points of Presence — hundreds worldwide)
    ↓ (edge serves users; pulls from regional cache on miss)
End User
```

**Cache population strategy: Pull (lazy loading)**
- A video segment is not pushed to all CDN nodes upfront.
- On the first request for a segment at an edge node, the edge fetches from the regional cache (or origin if regional also misses) and caches it.
- Subsequent requests for the same segment at the same edge are served from cache with no origin round-trip.
- Hot videos (millions of viewers) quickly saturate all edge caches. Cold videos (rarely watched) are only cached on the small number of edges that see traffic for them.

**Cache hit ratios:**
- A video in the top 0.1% by popularity might have 99%+ cache hit rate.
- Long-tail videos (most videos are rarely watched) have low hit rates — these are served from origin or regional cache.
- YouTube reports ~80% of its traffic is served from CDN edge without hitting the origin data center.

**Caching headers on segments:**
```
Cache-Control: public, max-age=31536000   (1 year — immutable segments; URL contains hash)
```

Video segments are content-addressed (URL contains a hash of the content). They never change in place. This allows very long TTLs without stale content risk.

---

**Q: How do you store and serve video metadata, view counts, and comments?**

**Video metadata (title, description, tags, uploader, thumbnail URL):**
- Stored in a relational database (MySQL / Spanner at Google's scale).
- Read-heavy; frequently cached in Redis or Memcached.
- Indexed for search (Elasticsearch or a dedicated search service).

**View count:**
- Naive approach: `UPDATE videos SET view_count = view_count + 1 WHERE id = ?`
  - At 41.7M concurrent viewers, this is millions of writes/sec to a single row per video — a massive write contention hot-spot.
- **Correct approach: approximate counting with eventual consistency**
  - Each app server maintains in-memory counters per video.
  - Periodically (every 30s or every 1,000 views), flush counts to a distributed counter store or Kafka.
  - A counter aggregation job rolls up to the DB.
  - YouTube shows approximate counts ("1.2M views" not "1,203,847 views") — this is intentional and tolerates eventual consistency.

**Likes:**
- Same approach: write to Kafka, aggregate asynchronously.

**Comments:**
- Append-only writes. Store in a horizontally sharded database (MySQL with `video_id` sharding, or Cassandra).
- Comments are not globally ordered — within a video's comments, sort by time or likes.
- Top-level comments and nested replies. Fetch top N comments on page load; load more on scroll.

---

**Q: How does YouTube's search work at high level?**

Search is a subsystem that could be its own case study. At a high level:

**Indexing pipeline:**
```
New video published (video_ready event from Kafka)
    ↓
Indexing Service
    ↓
Elasticsearch / custom inverted index
    ↓
Search queries hit the index
```

**What gets indexed:**
- Title, description, tags (highest weight on title)
- Auto-generated captions / transcripts (enables searching spoken words in videos)
- Channel name, category

**Ranking factors (simplified):**
- Text relevance (BM25 or learned-sparse retrieval)
- View count, likes, engagement rate
- Freshness (recency of upload)
- Personalisation (user's watch history — ML-based)

**At YouTube's scale:**
- The search index contains metadata for 800M+ videos.
- The index must support ~10M search queries/day.
- Elasticsearch at this scale requires sharding the index across hundreds of nodes.
- YouTube (Google) uses custom search infrastructure layered on top of Google's search technology.

---

**Q: How do recommendations work?**

Recommendations are a two-stage pipeline: **candidate generation** then **ranking**.

**Stage 1: Candidate Generation (recall)**
- Goal: narrow down from 800M videos to a few hundred candidate videos for this user.
- ML model (collaborative filtering or two-tower neural network):
  - Input: user's watch history, search history, demographics.
  - Output: ~hundreds of candidate video IDs.
- Approximate nearest-neighbour (ANN) search: embed each video and user in a vector space; find videos close to the user's embedding (FAISS, ScaNN).

**Stage 2: Ranking (precision)**
- Goal: rank the ~300 candidates by probability of being watched and engaged with.
- A more expensive ML model scores each candidate.
- Features: video freshness, watch time on similar videos, CTR of the thumbnail, user's historical engagement with the channel.

**Infrastructure:**
- Candidate generation runs offline / near-line (updated every few minutes).
- Ranking runs online, at request time (< 100ms budget for the ranking model).
- User embeddings and video embeddings are pre-computed and stored in a vector store.
- Recommendation results are cached per user and refreshed periodically.

---

**Q: What does the full system architecture look like?**

```
Uploader
    ↓ (chunked resumable upload)
Upload Service → Raw Video Store (GCS/S3)
    ↓ (Kafka: video_uploaded)
Transcoding Workers (horizontal fleet)
    ↓
Transcoded Video Store (GCS/S3) → CDN (push/pull)
    ↓ (Kafka: video_ready)
Indexing Service → Search Index (Elasticsearch)
    ↓
Metadata DB (MySQL/Spanner) ← Video metadata, user data

Viewer
    ↓
CDN Edge Node (serves 80% of requests)
    ↓ (cache miss)
CDN Regional Cache
    ↓ (cache miss)
Origin Storage (GCS/S3)

View counts / Likes:
Viewer action → App Server (in-memory counter) → Kafka → Counter Aggregator → DB

Comments:
Viewer → Comment Service → Cassandra (sharded by video_id)

Recommendations:
Background job → Candidate Gen (ANN on embeddings) → Ranker → Redis (cache per user)
    → Served at page load from cache
```

**Key decisions summarised:**

| Decision | Choice | Reason |
|----------|--------|--------|
| Video upload | Chunked + resumable | Handles large files, network interruptions |
| Transcoding | Async, parallel workers via Kafka | Decouples upload from processing; scalable |
| Streaming format | HLS / DASH with segments | ABR, CDN-cacheable, independent chunks |
| CDN strategy | 3-tier pull model | Global low-latency delivery, 80% origin offload |
| View counts | In-memory + Kafka aggregation | Avoids DB hot-spot at millions of writes/sec |
| Metadata | MySQL + Redis cache | Relational integrity + low read latency |
| Recommendations | Two-stage candidate gen + ranking | Recall vs precision separation; scalable |