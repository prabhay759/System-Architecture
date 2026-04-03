# 🏗️ System Design of the Day — Day 2
# Distributed Cache (Redis / Memcached)

> **Difficulty:** ⭐⭐⭐⭐ Medium-Hard  
> **Asked at:** Google, Meta, Amazon, Microsoft, Uber, Netflix  
> **Tags:** `#cache` `#redis` `#distributed-systems` `#consistency` `#scalability`

---

## 📌 Problem Statement

Design a **distributed caching system** like Redis or Memcached that:
- Stores key-value pairs in memory for ultra-fast reads
- Scales horizontally across many nodes
- Handles node failures gracefully without data loss
- Supports TTL-based expiration and eviction policies

---

## 🎯 Functional Requirements

| # | Requirement |
|---|-------------|
| 1 | `set(key, value, ttl)` — store a value with optional expiry |
| 2 | `get(key)` — retrieve value in < 1ms |
| 3 | `delete(key)` — remove a key explicitly |
| 4 | TTL-based auto expiration |
| 5 | Support for common data structures: String, Hash, List, Set, Sorted Set |

---

## 🚫 Non-Functional Requirements

- **Latency:** P99 read/write < 1ms
- **Availability:** 99.99% (no single point of failure)
- **Scalability:** Handle millions of ops/sec across nodes
- **Consistency:** Eventual consistency acceptable (tunable)
- **Durability:** Optional — pure cache can be ephemeral

---

## 📐 Capacity Estimation

```
Operations:  10M reads/sec + 1M writes/sec
Key size:    avg 100 bytes
Value size:  avg 1 KB
Hot dataset: 200 GB (fits in memory across cluster)

Nodes needed (64 GB RAM each):
  200 GB / (64 GB × 0.75 usable) ≈ 5 nodes (add 1 replica each = 10 total)

Throughput per node:
  Redis single-threaded: ~100K ops/sec
  10M ops / 100K = 100 shards → use 10 nodes × 10 threads via Redis Cluster
```

---

## 🔑 Core Design Decisions

### 1. How to Distribute Keys Across Nodes?

#### ❌ Naive Modulo Hashing
```
node = hash(key) % N
```
**Problem:** When N changes (node added/removed), almost all keys remap → **cache stampede**.

---

#### ✅ Consistent Hashing (Used by Redis Cluster, Memcached)
```
Hash ring: 0 ──────────────────────────── 2³²

  Node A owns: [0 → 85M)
  Node B owns: [85M → 170M)
  Node C owns: [170M → 256M)

  hash("user:123") = 100M → goes to Node B
```
- Adding/removing a node only remaps **~1/N keys** (not all)
- **Virtual nodes (vnodes):** Each physical node has 150+ virtual positions → even distribution

---

### 2. Replication Strategy

```
Primary ──writes──► Replica 1
         ──writes──► Replica 2

Reads can go to Replica (eventual consistency)
or Primary only (strong consistency)
```
- **Async replication** (default Redis) — fast writes, tiny window of data loss
- **Sync replication** — no data loss, higher write latency

---

### 3. Cache Eviction Policies

| Policy | Behavior | Best For |
|--------|----------|----------|
| **LRU** (Least Recently Used) | Evict least recently accessed key | General purpose |
| **LFU** (Least Frequently Used) | Evict least accessed over time | Skewed access patterns |
| **TTL-based** | Evict expired keys first | Session data, tokens |
| **Random** | Evict a random key | When access pattern is uniform |
| **No eviction** | Return error when full | Critical data only |

> 💡 Redis default: `allkeys-lru` — safe for most use cases.

---

## 🏛️ High-Level Architecture

```
                    ┌──────────────────────────────────────────┐
                    │               Clients                    │
                    │   (App Servers / Microservices)          │
                    └───────────┬──────────────────────────────┘
                                │
                    ┌───────────▼──────────────────────────────┐
                    │         Cache Client Library             │
                    │  (handles routing, retries, failover)    │
                    │  e.g. Jedis, ioredis, StackExchange.Redis│
                    └───────────┬──────────────────────────────┘
                                │  Consistent Hash → pick shard
              ┌─────────────────┼─────────────────┐
              │                 │                 │
   ┌──────────▼──────┐ ┌────────▼───────┐ ┌──────▼──────────┐
   │   Shard 1       │ │   Shard 2      │ │   Shard 3       │
   │  ┌───────────┐  │ │ ┌───────────┐  │ │ ┌───────────┐   │
   │  │ Primary   │  │ │ │ Primary   │  │ │ │ Primary   │   │
   │  │ (Redis)   │  │ │ │ (Redis)   │  │ │ │ (Redis)   │   │
   │  └─────┬─────┘  │ │ └─────┬─────┘  │ │ └─────┬─────┘   │
   │        │ async  │ │       │ async  │ │       │ async   │
   │  ┌─────▼─────┐  │ │ ┌─────▼─────┐  │ │ ┌─────▼─────┐   │
   │  │ Replica   │  │ │ │ Replica   │  │ │ │ Replica   │   │
   │  └───────────┘  │ │ └───────────┘  │ │ └───────────┘   │
   └─────────────────┘ └────────────────┘ └─────────────────┘
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                    ┌───────────▼──────────────────────────────┐
                    │        Sentinel / Cluster Manager        │
                    │  - Monitors primary health               │
                    │  - Auto-promotes replica on failure      │
                    │  - Notifies clients of topology changes  │
                    └──────────────────────────────────────────┘
                                │
                    ┌───────────▼──────────────────────────────┐
                    │         Backing Store (DB)               │
                    │      (on cache miss, read-through)       │
                    └──────────────────────────────────────────┘
```

---

## 🔄 Caching Patterns

### Pattern 1 — Cache Aside (Lazy Loading) ✅ Most Common
```
READ:
  1. App checks cache → HIT → return value
  2. Cache MISS → App queries DB → stores result in cache → returns value

WRITE:
  1. App writes to DB
  2. App invalidates (deletes) cache key
  3. Next read will repopulate cache
```
✅ Cache only contains requested data  
❌ First request is always slow (cold start)  
❌ Risk of stale data between write and invalidation

---

### Pattern 2 — Write Through
```
WRITE:
  1. App writes to cache
  2. Cache synchronously writes to DB
  3. Return success only after both succeed
```
✅ Cache always consistent with DB  
❌ Higher write latency (two writes)  
❌ Cache fills with data that may never be read

---

### Pattern 3 — Write Behind (Write Back)
```
WRITE:
  1. App writes to cache only → return immediately
  2. Cache asynchronously flushes to DB in batches
```
✅ Fastest writes  
❌ Risk of data loss if cache crashes before flush  
❌ Complex implementation

---

### Pattern 4 — Read Through
```
READ:
  1. App always reads from cache
  2. On MISS — cache itself fetches from DB and populates
  3. App never talks to DB directly
```
✅ Simpler app code  
❌ First read is always slow

---

## 🗄️ Redis Data Structures & Use Cases

```
String    → Session tokens, counters, feature flags
           SET user:session:abc "data"  EX 3600

Hash      → User profile objects
           HSET user:123 name "Alice" age 30 city "NY"

List      → Recent activity feed, queues
           LPUSH notifications:123 "You got a like"
           LRANGE notifications:123 0 9  (last 10)

Set       → Unique visitors, tags, friend lists
           SADD page:views:home "user:1" "user:2"
           SCARD page:views:home  → count

Sorted Set → Leaderboards, rate limiting
           ZADD leaderboard 9500 "alice"
           ZREVRANGE leaderboard 0 9  → top 10

Bitmap    → Feature flags per user ID (1 bit/user!)
           SETBIT feature:dark_mode 12345 1

HyperLogLog → Unique count estimation with tiny memory
           PFADD unique_visitors "user:abc"
           PFCOUNT unique_visitors → ~100 (±0.8% error)
```

---

## 🚀 Failure Scenarios & Solutions

| Scenario | Problem | Solution |
|----------|---------|----------|
| **Cache Stampede** | Cache expires → 10K requests hit DB simultaneously | **Probabilistic early expiration** or **mutex lock** on first miss |
| **Hot Key Problem** | Single key (e.g. celebrity profile) overwhelms one shard | **Local in-process cache** (L1) + distribute reads across replicas |
| **Cold Start** | Cache is empty after restart → DB overwhelmed | **Cache warming** — pre-populate on startup from DB or snapshot |
| **Cache Penetration** | Requests for non-existent keys bypass cache, hammer DB | **Bloom filter** at cache layer — reject keys that don't exist in DB |
| **Network Partition** | Cache and DB split-brain | Use **circuit breaker** — serve stale data or degrade gracefully |
| **Primary Failure** | Shard goes down | **Sentinel** auto-promotes replica in ~30 seconds |

---

## 📊 Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data Structures | Rich (String, Hash, List, Set, ZSet) | String only |
| Persistence | RDB snapshots + AOF log | ❌ None |
| Replication | ✅ Primary-Replica | ❌ None (client-side only) |
| Clustering | ✅ Redis Cluster (built-in) | Client-side sharding only |
| Pub/Sub | ✅ Yes | ❌ No |
| Lua Scripting | ✅ Yes | ❌ No |
| Memory Efficiency | Slightly higher overhead | More memory-efficient for plain strings |
| **Winner** | 🏆 General purpose | ✅ Simple high-speed string cache only |

---

## 🛡️ Persistence Options (Redis)

```
RDB (Redis Database Snapshot):
  - Point-in-time snapshot to disk every N seconds
  - Fast restart, compact file
  - Risk: lose data since last snapshot

AOF (Append-Only File):
  - Logs every write command
  - fsync every second (default) or every command
  - Larger files, slower restart
  - Near-zero data loss

Hybrid (Recommended for production):
  - Use RDB for fast restarts + AOF for durability
  - Best of both worlds
```

---

## 🎙️ Interview Tips

> **Q: How do you handle cache invalidation?**  
> A: It depends on consistency needs. For most cases, use **Cache Aside with TTL** — accept brief staleness. For strict consistency, use **write-through** or **event-driven invalidation** via CDC (Change Data Capture) on the DB.

> **Q: What is a cache stampede and how do you prevent it?**  
> A: When a popular key expires simultaneously for many users, they all miss and hit the DB at once. Solutions: (1) **Mutex/lock** — only one request fetches from DB, rest wait. (2) **Probabilistic early expiration** — randomly re-fetch before key actually expires. (3) **Background refresh** — always serve stale, refresh asynchronously.

> **Q: How does Redis Cluster handle resharding?**  
> A: Redis Cluster uses 16,384 hash slots. Each node owns a range. During resharding, slots migrate one-by-one with **MIGRATE** command while serving live traffic. Clients use **ASK/MOVED** redirects to find the right node.

---

## 📊 Technology Choices Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Cache Engine** | Redis 7.x | Rich data structures, cluster mode, persistence |
| **Cluster Mode** | Redis Cluster | Built-in sharding + failover |
| **Client Library** | ioredis / Jedis | Cluster-aware, connection pooling |
| **Monitoring** | Redis Sentinel + Prometheus | Health checks + metrics |
| **Eviction** | allkeys-lru | Safe default for general caches |
| **Persistence** | RDB + AOF hybrid | Balance between performance and durability |

---

## 📚 Further Reading
- [Redis Architecture — Official Docs](https://redis.io/docs/management/replication/)
- [Consistent Hashing explained — Tom White](https://tom-e-white.com/2007/11/consistent-hashing.html)
- [How Twitch uses Redis for chat](https://blog.twitch.tv/en/2015/12/18/twitch-engineering-an-introduction-and-overview-1a00023c310/)
- [Facebook TAO — Distributed Cache at Scale](https://engineering.fb.com/2013/06/25/core-data/tao-the-power-of-the-graph/)

---

*Posted as part of the **Architecture of the Day** series.*  
*Previous: [Day 01 — URL Shortener](./day-01-url-shortener.md) | Next: Day 03 — News Feed / Timeline*
