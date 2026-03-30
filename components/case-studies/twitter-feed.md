# Case Study: Twitter Feed (News Feed / Timeline)

---

**Q: What are the requirements for a Twitter-like feed system?**

**Functional requirements:**
- Users can post tweets (≤ 280 characters, optionally with media)
- Users can follow other users
- Each user has a home timeline: a reverse-chronological feed of tweets from people they follow
- Users can view a profile timeline: all tweets from a specific user
- Likes, retweets, replies (interactions)
- Search tweets (separate subsystem; excluded here)

**Non-functional requirements:**
- **Write latency**: Posting a tweet should feel instant (< 200ms p99)
- **Read latency**: Loading the home timeline should be fast (< 100ms p99)
- **Eventual consistency**: Seeing a tweet 1–2 seconds after it's posted is acceptable
- **Scale**: 300M daily active users, 500M tweets/day, each user follows ~200 accounts on average
- **Availability**: Reads must be highly available; brief write unavailability is tolerable

---

**Q: How do you estimate scale for a Twitter feed?**

**Writes (tweets posted):**
- 500M tweets/day = 500M / 86400 ≈ **5,800 tweets/sec** average.
- Peak (2–3× average): ~15,000 tweets/sec.

**Reads (timeline loads):**
- 300M DAU, each opens the app 5× per day and loads ~20 tweets per open.
- 300M × 5 = 1.5B timeline loads/day = 1.5B / 86400 ≈ **17,000 timeline reads/sec**.
- Plus tweet content fetches: 17,000 × 20 tweets = 340,000 tweet reads/sec.

**Fan-out (the core challenge):**
- When a user posts a tweet, it must be "delivered" to all their followers' timelines.
- A user with 1M followers posting once generates 1M timeline write operations.
- 5,800 tweets/sec × avg 200 followers = **1.16M fan-out writes/sec** (for non-celebrities).
- Celebrities (10M+ followers) create massive spikes.

**Storage:**
- Tweet: id (8B) + user_id (8B) + text (280 chars = 280B) + timestamp (8B) + metadata ≈ 500 bytes.
- 500M tweets/day × 500 bytes = 250 GB/day of tweet data.
- 5 years = ~450 TB of tweet text. Use object storage + columnar DB for old data.

---

**Q: What are the two approaches to serving a home timeline — fan-out on write vs fan-out on read?**

This is the central design choice for any feed system.

**Fan-out on write (push model):**

When a user posts a tweet, immediately "push" the tweet ID into every follower's pre-computed timeline cache.

```
User posts tweet → Fanout Service → for each follower: prepend tweet_id to their timeline in Redis
```

- Timeline reads are O(1): just read the pre-built list from Redis.
- Write amplification: 1 tweet → N writes (N = follower count).
- Works well for users with a manageable follower count (< ~10k).
- Fails for celebrities: Lady Gaga has 50M followers. One tweet = 50M Redis writes. That's a write storm.

**Fan-out on read (pull model):**

When a user opens their feed, dynamically compute the timeline by fetching recent tweets from each account they follow.

```
User opens feed → Fetch list of 200 accounts they follow
               → For each, fetch their 20 most recent tweets
               → Merge and sort by timestamp
               → Return top 20
```

- Write path is cheap: just write the tweet once.
- Read path is expensive: 200 DB/cache queries merged per timeline load.
- At 17,000 timeline loads/sec × 200 fan-in reads = 3.4M read operations/sec — costly.
- High latency on read: merging 200 sources + sorting takes time.

**Hybrid approach (what Twitter actually uses):**

- **Fan-out on write** for users with ≤ ~10,000 followers. Their tweets get pushed into followers' timeline caches immediately.
- **Fan-out on read** for celebrities (> ~10,000 followers, "mega-influencers"). Their tweets are NOT pre-distributed. Instead, when you load your timeline, the system checks if you follow any celebrities, fetches their recent tweets separately, and merges them in at read time.

This bounds write amplification while keeping read latency low for the common case.

---

**Q: How do you implement the timeline cache in Redis?**

Each user's home timeline is stored in Redis as a **sorted set** keyed by `timeline:{user_id}`, with the tweet's timestamp as the score and the tweet ID as the member.

```
Key: timeline:12345
Type: Sorted Set (ZSET)
Members: tweet_id (as string)
Score: unix_timestamp (for sort order)
```

**Operations:**
- **Fan-out write:** `ZADD timeline:{follower_id} {timestamp} {tweet_id}` — O(log N)
- **Timeline read:** `ZREVRANGE timeline:{user_id} 0 19` — top 20 by score, O(log N + 20)
- **Trim to cap size:** `ZREMRANGEBYRANK timeline:{user_id} 0 -801` — keep newest 800, evict older

**Why cap at ~800 entries?**
- Storing the full history per user (potentially millions of tweets) consumes too much Redis memory.
- In practice, users rarely scroll beyond a few hundred entries before refreshing.
- Cap at 800 per user. Deeper scrolling falls back to a DB query.

**Memory estimate:**
- 300M users × 800 entries × ~16 bytes per ZSET entry = **~3.8 TB**.
- Use Redis Cluster (horizontal sharding by `user_id`). 100 nodes × 40 GB RAM = 4 TB capacity.

---

**Q: Where do you store tweet content, and how does a timeline read hydrate the tweets?**

The timeline cache stores only **tweet IDs** — not the full text, media URLs, like counts, etc. This is intentional:

- Keeps the timeline cache lean (IDs only ≈ 8 bytes each vs 500 bytes per full tweet).
- Tweet content is fetched separately from a **tweet store** by ID.
- Allows the timeline cache to be invalidated/updated without rewriting content.

**Tweet store options:**

| Store | Use case |
|-------|---------|
| MySQL / PostgreSQL | Source of truth for tweet content; ACID for creates |
| Redis (tweet cache) | Cache tweet objects by ID; TTL = 24–48h |
| Cassandra | High-volume append-only tweet writes; wide-column optimised for `tweet_id` lookups |

**Timeline hydration — the read path:**

```
1. ZREVRANGE timeline:{user_id} 0 19 → [tweet_id_1, tweet_id_2, ..., tweet_id_20]
2. MGET tweet:{id} for all 20 IDs (batch Redis read — one round trip via pipeline)
3. If cache miss on any tweet_id → read from DB
4. Merge celebrity tweets (fan-on-read for users following mega-influencers)
5. Sort combined list by timestamp, return top 20
```

Step 2 uses Redis pipeline to batch 20 GETs into one network round trip — crucial for latency.

---

**Q: How does the fan-out service work? Walk through the architecture.**

```
Tweet POST request
        ↓
   Write Service
        ↓
1. Write tweet to DB (Cassandra or MySQL)
2. Write tweet to tweet cache (Redis)
3. Publish tweet_id to Kafka topic: "new_tweets"
        ↓
   Fanout Workers (Kafka consumers, horizontally scaled)
        ↓
4. Look up follower list for tweet author from Follower Service
5. For each follower (if non-celebrity):
      ZADD timeline:{follower_id} {timestamp} {tweet_id}
6. Trim each timeline to 800 entries
7. (Skip fan-out for celebrity authors — handled at read time)
```

**Why Kafka in the middle?**
- Decouples the write request from the fan-out work. The user's POST returns after step 3 (< 50ms); fan-out happens asynchronously.
- Kafka provides durability: if fan-out workers are slow or restart, no tweet deliveries are lost.
- Workers are horizontally scalable independently of the write service.

**Fan-out latency:**
- Non-celebrity (200 followers): fan-out completes in < 1 second.
- Popular user (100k followers): fan-out completes in seconds to tens of seconds.
- Celebrity (50M followers): fan-out is skipped; handled at read time.

---

**Q: How do you store and serve the follower/following graph?**

The social graph is a massive directed graph: edge `A → B` means "A follows B."

**Scale:**
- 300M users, avg 200 followers each = ~60B directed edges.
- 60B edges × ~16 bytes/edge = ~960 GB. Too large for a single DB.

**Storage options:**

**MySQL / PostgreSQL (source of truth):**
```sql
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    created_at  TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_followee ON follows(followee_id);
```
- Lookup "who follows X" (for fan-out): `SELECT follower_id WHERE followee_id = X`
- Lookup "who does X follow" (for read-time fan-in): `SELECT followee_id WHERE follower_id = X`

**Redis (hot cache):**
- For active users, cache their follower list as a Redis Set or sorted set.
- `followers:{user_id}` → Set of follower user_ids.
- Only cache the "following" list (smaller, needed at read time) — not the full follower list for celebrities.

**Dedicated graph database:**
- At Twitter's scale, graph traversal (mutual follows, recommendations) benefits from Neo4j or a custom solution.
- Twitter built **FlockDB** — a distributed graph database optimised for social graph edge storage, supporting fast paginated reads of edges.

---

**Q: How do you handle the "celebrity problem" (users with millions of followers)?**

A celebrity posting a single tweet would require 50M Redis ZADD operations — a write storm that would delay fan-out for normal users by minutes.

**Solution: Identify celebrities, skip their fan-out, inject at read time.**

**Celebrity threshold:** Twitter uses ~10,000 followers as the cutoff. Above this, a user is flagged as a "celebrity" in the user metadata.

**Read-time injection:**
```
1. Load timeline from Redis (fan-out-on-write entries)
2. Look up the user's following list
3. Filter for any followed celebrities (flagged users)
4. For each celebrity followed: fetch their 20 most recent tweets from tweet cache
5. Merge celebrity tweets into the timeline
6. Deduplicate, re-sort by timestamp, return top 20
```

**Caching celebrity timelines:**
- A celebrity's own recent tweets (last 100) are cached in a single Redis key: `user_timeline:{celebrity_id}`.
- This cache is shared across all followers — 50M followers all hit the same cache key.
- One cache entry serves all fans vs. 50M individual timeline entries.

**Trade-off:**
- Read latency increases slightly (extra merge step).
- But write latency becomes bounded and predictable regardless of follower count.

---

**Q: How do you handle timeline pagination and infinite scroll?**

Users scroll down and expect to see older tweets on demand.

**Cursor-based pagination (not offset-based):**

Use the tweet ID (or timestamp) as a cursor. The client sends the ID of the oldest tweet it has seen, and the server returns the next page of older tweets.

```
GET /timeline?before_id=1234567890&limit=20
→ Returns 20 tweets with id < before_id, sorted descending
```

**Why not offset pagination?**
- `LIMIT 20 OFFSET 1000` is O(N) in most databases — DB must scan 1,000 rows to skip them.
- With cursor: `WHERE tweet_id < cursor ORDER BY tweet_id DESC LIMIT 20` — uses the index efficiently, O(log N + 20).
- New tweets posted at the top don't shift offsets and cause duplicate/skipped items.

**Falling off the Redis cache:**
- The Redis timeline is capped at ~800 entries (covering roughly the last few days for active followees).
- If a user scrolls past the cached entries, fall back to the DB:
  - Fetch `follows` for the user, get all `followee_id`s.
  - `SELECT * FROM tweets WHERE user_id IN (...) AND created_at < cursor ORDER BY created_at DESC LIMIT 20`
  - This is expensive at scale — acceptable because deep scrolling is rare.

---

**Q: What does the full system architecture look like?**

```
Client (iOS/Android/Web)
         ↓
    CDN (static assets, media) + Load Balancer
         ↓
   API Gateway (auth, rate limiting)
    ↙              ↘
Write Service      Read Service
    ↓                   ↓
 Cassandra         Redis Cluster (timeline cache)
 (tweet store)     Redis Cluster (tweet content cache)
    ↓                   ↓  (cache miss)
  Kafka           Cassandra / MySQL (tweet store)
    ↓
Fanout Workers
    ↓
Redis Cluster (write to follower timelines)
    ↑
Follower Service
    ↑
MySQL (social graph) + Redis (hot follower lists)

Media: separate upload → S3 → CDN
Analytics: Kafka → Flink → Druid (real-time counters)
```

**Key decisions summarised:**

| Decision | Choice | Reason |
|----------|--------|--------|
| Timeline storage | Redis sorted sets | O(1) read, sub-ms latency |
| Tweet storage | Cassandra | High write throughput, append-only workload |
| Fan-out strategy | Hybrid push/pull | Bounds write amplification for celebrities |
| Fan-out decoupling | Kafka | Async delivery, durable, scalable workers |
| Pagination | Cursor-based | O(log N) DB reads, no duplicates on new tweets |
| Social graph | MySQL + Redis | SQL for integrity, Redis for hot follower lists |