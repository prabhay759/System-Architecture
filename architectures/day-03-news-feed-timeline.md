# 🏗️ System Design of the Day — Day 3
# News Feed / Timeline (Instagram / Twitter / Facebook)

> **Difficulty:** ⭐⭐⭐⭐⭐ Hard  
> **Asked at:** Meta, Twitter/X, Google, LinkedIn, Snap, TikTok  
> **Tags:** `#newsfeed` `#fanout` `#timeline` `#distributed-systems` `#scalability`

---

## 📌 Problem Statement

Design a **news feed system** like Instagram, Facebook, or Twitter that:
- Shows a personalized, ranked feed of posts from people a user follows
- Handles users with vastly different follower counts (celebrities vs regular users)
- Serves feeds with low latency at massive scale
- Supports likes, comments, reposts, and real-time updates

---

## 🎯 Functional Requirements

| # | Requirement |
|---|-------------|
| 1 | Users can create posts (text, images, video) |
| 2 | Users can follow / unfollow other users |
| 3 | User sees a ranked feed of posts from followed users |
| 4 | Feed is paginated (cursor-based, infinite scroll) |
| 5 | Likes, comments, reposts are supported |
| 6 | Feed updates in near real-time |

---

## 🚫 Non-Functional Requirements

- **Latency:** Feed load < 200ms (P99)
- **Scale:** 500M DAU, 5M posts/day, 1B feed reads/day
- **Availability:** 99.99% — feed must always be readable
- **Consistency:** Eventual — slight delay in feed updates is acceptable
- **Ordering:** Ranked (relevance + recency), not purely chronological

---

## 📐 Capacity Estimation

```
Posts per day:     5M posts/day  →  ~58 posts/sec
Feed reads/day:    1B reads/day  →  ~11,600 reads/sec
Avg follows/user:  300
Fanout writes:     58 posts/sec × 300 followers = ~17,400 writes/sec

Storage per post:
  post_id (8B) + user_id (8B) + content (500B) + media_url (100B) + ts (8B)
  ≈ 624 bytes per post
  5M posts/day × 624B ≈ 3 GB/day → ~1 TB/year (posts only)

Feed cache per user:
  top 500 post IDs × 8 bytes = 4 KB per user
  500M users × 4 KB = 2 TB total (store only active users ~10% = 200 GB)
```

---

## 🔑 The Core Design Problem — Fanout

> When a user posts, how do you get that post into all their followers' feeds?

### Option A — Fanout on Write (Push Model) ✅
```
User posts
    → Write post to Posts DB
    → Fanout Service reads follower list
    → Writes post ID into each follower's feed cache

Result: Feed reads are instant (pre-computed)
Cost:   Expensive writes for celebrities (Kylie Jenner → 400M followers)
```

### Option B — Fanout on Read (Pull Model)
```
User opens app
    → Fetch list of N followed users
    → Query Posts DB for latest posts from each
    → Merge & rank results

Result: Simple writes, complex & slow reads
Cost:   O(N) DB queries per feed load — unscalable at 500M DAU
```

### Option C — Hybrid Model ✅✅ (Used by Instagram, Twitter)
```
Regular users (< 1M followers) → Fanout on WRITE (push to feed cache)
Celebrities   (> 1M followers) → Fanout on READ  (pull at request time)

Feed load:
  1. Read pre-computed feed from cache (covers ~95% of posts)
  2. Merge with live-fetched posts from followed celebrities
  3. Rank & return
```
> 💡 This is the key insight interviewers want to hear.

---

## 🏛️ High-Level Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                          Clients                                │
  └────────────┬───────────────────────────────┬────────────────────┘
               │ POST /post                    │ GET /feed
               ▼                               ▼
  ┌────────────────────────┐     ┌─────────────────────────────────┐
  │     Post Service       │     │         Feed Service            │
  │  - Validate & store    │     │  - Fetch from feed cache        │
  │  - Upload media to S3  │     │  - Merge celebrity posts        │
  │  - Publish to Kafka    │     │  - Rank & paginate              │
  └────────────┬───────────┘     └──────────────┬──────────────────┘
               │                                │
               ▼                                ▼
  ┌────────────────────────┐     ┌──────────────────────────────────┐
  │   Kafka (Post Events)  │     │     Feed Cache (Redis)           │
  │   topic: new-posts     │     │   user_id → [post_id, post_id…]  │
  └────────────┬───────────┘     │   sorted by rank score           │
               │                 └──────────────┬───────────────────┘
               ▼                                │ cache miss
  ┌────────────────────────┐                    ▼
  │    Fanout Service      │     ┌──────────────────────────────────┐
  │  - Consume post events │     │       Posts DB (Cassandra)       │
  │  - Read follower list  │     │   post_id, user_id, content,     │
  │  - Push to feed caches │     │   media_url, created_at, likes   │
  │    (skip celebrities)  │     └──────────────────────────────────┘
  └────────────┬───────────┘
               │                 ┌──────────────────────────────────┐
               ▼                 │       Graph DB / Follow DB       │
  ┌────────────────────────┐     │  (user_id → [follower_ids])      │
  │  Follow DB (MySQL)     │◄────│  sharded by user_id              │
  └────────────────────────┘     └──────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │                     Media Pipeline                               │
  │  Post Service → S3 → Lambda → Transcode/Resize → CDN (CloudFront)│
  └──────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Request Flows

### ✍️ Create a Post
```
1. Client POSTs { content, media } to Post Service
2. Post Service validates, assigns post_id (Snowflake), stores in Posts DB
3. Media uploaded to S3, async transcoded, served via CDN
4. Post event published to Kafka: { post_id, author_id, timestamp }
5. Fanout Service consumes event:
   a. Fetches author's follower list from Follow DB
   b. If follower count < 1M → push post_id to each follower's feed cache (Redis LPUSH)
   c. If author is celebrity → skip (pull at read time)
6. Notification Service (separate consumer) sends push notifications
```

### 📖 Load Feed
```
1. Client requests GET /feed?cursor=<last_post_id>&limit=20
2. Feed Service fetches user's pre-computed feed list from Redis
   → [post_id_1, post_id_2, ... post_id_500]
3. Merge with live posts from celebrities user follows (direct DB query)
4. Batch-fetch post details from Posts DB (or post cache)
5. Ranking Service scores posts (recency + engagement + affinity)
6. Return ranked, paginated list to client
```

---

## 🏆 Feed Ranking

Real feeds are not purely chronological — they're ranked. Factors:

```
Score = w1×Recency + w2×EngagementRate + w3×UserAffinity + w4×ContentType

Recency:         time_decay(post.created_at)
EngagementRate:  (likes + comments×2 + shares×3) / impressions
UserAffinity:    how often you interact with this author
ContentType:     video > image > text (platform-specific weight)
```

At scale: use an **ML ranking model** (logistic regression → neural network) trained on historical engagement. Instagram uses a two-stage approach:
1. **Candidate generation** — retrieve top 500 posts from feed cache
2. **Ranking** — ML model scores and re-ranks top 500 → return top 25

---

## 🗄️ Database Schema

```sql
-- Posts table (Cassandra — wide column, high write throughput)
CREATE TABLE posts (
    post_id     BIGINT PRIMARY KEY,   -- Snowflake ID (sortable by time)
    user_id     BIGINT,
    content     TEXT,
    media_urls  LIST<TEXT>,
    like_count  COUNTER,
    created_at  TIMESTAMP
);

-- Follow graph (MySQL — relational, easy traversal)
CREATE TABLE follows (
    follower_id  BIGINT,
    followee_id  BIGINT,
    created_at   TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_followee ON follows(followee_id); -- reverse lookup

-- Feed cache (Redis - not DB schema)
-- KEY:   feed:{user_id}
-- TYPE:  Sorted Set
-- SCORE: rank_score (float)
-- MEMBER: post_id
ZADD feed:12345 1703001234.5 "post:987"
ZREVRANGE feed:12345 0 19  -- top 20 posts by score
```

---

## 🚀 Scaling Deep Dive

### Scaling the Fanout Service
- **Parallel workers:** Kafka consumer group — multiple Fanout workers in parallel
- **Batch writes:** Write to Redis pipeline (batch 100 feed updates per round-trip)
- **Rate limiting for celebrities:** Skip fanout > 1M followers; use pull model
- **Priority queues:** Verified accounts / paid promotions get priority fanout

### Scaling Feed Cache
- **Redis Cluster** — shard by `user_id`
- **Limit feed size:** Keep only top 500 post IDs per user (evict oldest)
- **TTL:** Expire inactive users' feed caches (no need to maintain for users inactive > 30 days)

### Scaling Posts DB
- **Cassandra** — partition by `user_id`, cluster by `post_id` (time-sorted)
- **Read replicas** for feed hydration
- **CDN** for media — CloudFront / Fastly serves images/videos

---

## 🛡️ Edge Cases & Failure Scenarios

| Scenario | Solution |
|----------|----------|
| **Celebrity posts 1 tweet** | Skip fanout; followers pull at read time via celebrity post cache |
| **User follows 10,000 accounts** | Cap feed cache size; trim oldest entries; rank aggressively |
| **Fanout Service is slow** | Feed reads still work from existing cache; new posts appear with small delay |
| **User deletes a post** | Soft-delete in Posts DB; filter deleted posts at read time before returning |
| **Feed cache cold start** | On first login / cache miss, rebuild feed from Posts DB for followed users |
| **Thundering herd on viral post** | Cache at CDN edge; counter updates via Redis INCR (atomic); async DB write |

---

## 📊 Technology Choices Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Posts DB** | Cassandra | High write throughput, time-series data, horizontal scale |
| **Follow Graph** | MySQL (sharded) | Relational queries, well-understood, easy joins |
| **Feed Cache** | Redis Cluster (Sorted Set) | Sub-ms reads, score-based ranking built-in |
| **Message Queue** | Kafka | Durable, high-throughput fanout event stream |
| **Media Storage** | S3 + CloudFront | Durable object store + global CDN |
| **Search** | Elasticsearch | Full-text post search |
| **Ranking** | ML Model (TensorFlow Serving) | Personalized relevance scoring |

---

## 🎙️ Interview Tips

> **Q: How would you handle a user with 500M followers posting?**  
> A: Pure fanout-on-write is impossible — 500M Redis writes would take hours. Instead, mark them as "celebrity". On feed load, pull their latest posts separately and merge into the pre-computed feed. Cache the celebrity's post list aggressively at the CDN layer.

> **Q: How do you keep the feed consistent if a post is deleted?**  
> A: Soft-delete the post (set `deleted_at`). During feed hydration, filter out deleted post IDs before returning to the client. Eventual consistency is acceptable — a brief window where deleted posts appear is tolerable.

> **Q: How would you implement pagination?**  
> A: Use **cursor-based pagination** (not offset). Cursor = encoded `(rank_score, post_id)` of the last seen item. This is stable under insertions and O(1) to resume vs. OFFSET which gets slower as pages increase.

---

## 📚 Further Reading
- [Instagram Engineering — How the Feed Works](https://instagram-engineering.com/types-of-queries-and-their-characteristics-b44c29f82cef)
- [Twitter — Timelines at Scale](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)
- [Meta — TAO: Facebook's Distributed Data Store for Social Graph](https://engineering.fb.com/2013/06/25/core-data/tao-the-power-of-the-graph/)
- [LinkedIn — Feed Architecture](https://engineering.linkedin.com/blog/2016/03/follow-feed--the-architecture-of-linkedin-s-updates-feed)

---

*Posted as part of the **Architecture of the Day** series.*  
*Previous: [Day 02 — Distributed Cache](./day-02-distributed-cache.md) | Next: Day 04 — Rate Limiter*
