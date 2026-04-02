# 🏗️ System Design of the Day — Day 1
# URL Shortener (Bit.ly / TinyURL)

> **Difficulty:** ⭐⭐⭐ Medium  
> **Asked at:** Google, Amazon, Meta, Uber, Twitter  
> **Tags:** `#system-design` `#scalability` `#hashing` `#caching` `#databases`

---

## 📌 Problem Statement

Design a URL shortening service like **bit.ly** or **TinyURL** that:
- Takes a long URL and returns a unique short URL (e.g., `https://short.ly/aB3xK9`)
- Redirects users to the original URL when they visit the short link
- Handles hundreds of millions of URLs and billions of redirects per day

---

## 🎯 Functional Requirements

| # | Requirement |
|---|-------------|
| 1 | Given a long URL, generate a unique short URL |
| 2 | Redirect short URL → original URL (with low latency) |
| 3 | Short links expire after a configurable TTL (default: 5 years) |
| 4 | Users can optionally provide a custom alias |
| 5 | Track click analytics (count, geo, device) |

---

## 🚫 Non-Functional Requirements

- **Availability:** 99.99% uptime (read-heavy system)
- **Latency:** Redirect < 10ms (P99)
- **Scale:** 500M new URLs/month, 100:1 read-to-write ratio
- **Uniqueness:** No two long URLs should map to the same short code (unless intentional)
- **Not required:** Real-time analytics (can be eventually consistent)

---

## 📐 Capacity Estimation

```
Writes:  500M URLs/month  →  ~200 URLs/sec
Reads:   50B redirects/month  →  ~20,000 redirects/sec

Storage per URL record:
  - short_code (7 chars)  =   7 bytes
  - long_url              = 200 bytes
  - created_at, ttl, user =  50 bytes
  - Total                 ≈ 257 bytes

Storage for 5 years:
  500M/month × 12 × 5 = 30 Billion URLs
  30B × 257 bytes ≈ 7.7 TB
```

---

## 🔑 Core Design Decision — Short Code Generation

This is the crux of the problem. Three main approaches:

### Option A — MD5/SHA256 Hash + Truncation
```
hash = MD5(long_url)  →  "5d41402abc4b2a76b9719d911017c592"
short_code = first 7 chars  →  "5d41402"
```
❌ **Problem:** Hash collisions. Two different URLs may produce the same 7-char prefix.

---

### Option B — Base62 Encoding of Auto-Increment ID ✅ (Recommended)
```
DB auto-increments ID:  12345678
Base62 encode(12345678) →  "8M0kX"  (pad to 7 chars → "008M0kX")
```
- Base62 alphabet: `[0-9][a-z][A-Z]`  → 62 characters
- 7 characters → 62^7 ≈ **3.5 Trillion** unique codes
- **No collision possible** — IDs are always unique

**Challenge:** Single DB for ID generation = bottleneck.  
**Solution:** Use a **Ticket Server** (dedicated ID generator service) or **Twitter Snowflake** IDs.

---

### Option C — Pre-generated Code Pool
A background worker pre-generates millions of unique codes and stores them in a **"used" / "unused"** table. Workers pull from the unused pool.

✅ No collision, fast  
❌ Operationally complex

---

## 🏛️ High-Level Architecture

```
                        ┌─────────────────────────────────────────┐
                        │              Clients                    │
                        └────────────┬───────────────┬────────────┘
                                     │               │
                             POST /shorten      GET /{code}
                                     │               │
                        ┌────────────▼───────────────▼────────────┐
                        │           Load Balancer (L7)            │
                        │         (AWS ALB / NGINX / Envoy)       │
                        └────────────┬───────────────┬────────────┘
                                     │               │
                        ┌────────────▼──┐       ┌────▼────────────┐
                        │  Write Service │       │ Redirect Service│
                        │  (Shorten API) │       │  (Read-heavy)   │
                        └───────┬───────┘       └──────┬──────────┘
                                │                      │
                   ┌────────────▼──────┐    ┌──────────▼──────────┐
                   │  ID Generator     │    │   Redis Cache        │
                   │  (Ticket Server   │    │   code → long_url    │
                   │   or Snowflake)   │    │   TTL: 24 hours      │
                   └────────────┬──────┘    └──────────┬──────────┘
                                │                      │ (cache miss)
                   ┌────────────▼──────────────────────▼──────────┐
                   │            Primary Database                   │
                   │        (PostgreSQL / MySQL / Cassandra)       │
                   │                                               │
                   │  Table: url_mapping                           │
                   │  ┌──────────┬───────────────┬─────────────┐  │
                   │  │ short_id │   long_url     │  expires_at │  │
                   │  ├──────────┼───────────────┼─────────────┤  │
                   │  │ 008M0kX  │ https://amaz.. │ 2030-01-01  │  │
                   │  └──────────┴───────────────┴─────────────┘  │
                   └───────────────────────┬───────────────────────┘
                                           │ Replication
                   ┌───────────────────────▼───────────────────────┐
                   │              Read Replicas (1–N)               │
                   └───────────────────────────────────────────────┘
                                           │ CDC / Kafka
                   ┌───────────────────────▼───────────────────────┐
                   │         Analytics Service (Async)              │
                   │     ClickHouse / BigQuery / Druid              │
                   └───────────────────────────────────────────────┘
```

---

## 🔄 Request Flows

### ✍️ Shorten a URL (Write Path)
```
1. Client sends POST /shorten { "url": "https://very-long-url.com/..." }
2. Write Service validates the URL
3. Calls ID Generator → gets unique integer ID (e.g., 12345678)
4. Base62 encodes → "008M0kX"
5. Stores in DB: { short_code: "008M0kX", long_url: "...", expires_at: ... }
6. Returns: { "short_url": "https://short.ly/008M0kX" }
```

### 🔀 Redirect a Short URL (Read Path)
```
1. Client visits https://short.ly/008M0kX
2. Redirect Service checks Redis Cache → HIT → return 301/302 redirect
   OR Cache MISS → query DB Read Replica → populate cache → redirect
3. Async: publish click event to Kafka → consumed by Analytics Service
```

### 301 vs 302 — Important Interview Point!
| Code | Meaning | Browser Caches? | Use When |
|------|---------|-----------------|----------|
| **301** | Permanent Redirect | ✅ Yes | Reduces server load, but you lose click analytics |
| **302** | Temporary Redirect | ❌ No | Every request hits your server — enables analytics tracking |

> 💡 **Answer:** Use **302** if analytics matter. Use **301** for pure performance.

---

## 🗄️ Database Schema

```sql
CREATE TABLE url_mapping (
    id           BIGINT PRIMARY KEY,          -- auto-increment / snowflake
    short_code   VARCHAR(10) NOT NULL UNIQUE, -- base62 encoded
    long_url     TEXT NOT NULL,
    user_id      BIGINT,                      -- nullable (anonymous allowed)
    custom_alias BOOLEAN DEFAULT FALSE,
    created_at   TIMESTAMP DEFAULT NOW(),
    expires_at   TIMESTAMP,
    click_count  BIGINT DEFAULT 0            -- or track in separate analytics table
);

CREATE INDEX idx_short_code ON url_mapping(short_code);
```

---

## 🚀 Scaling Deep Dive

### Scaling Reads (20K RPS)
- **Redis Cluster** in front of DB — cache hot URLs (20% of URLs = 80% of traffic)
- **CDN Edge Caching** — push popular redirects to CloudFront / Fastly edge nodes
- **Read Replicas** — horizontal scale for DB reads

### Scaling Writes (200 RPS → future 10K RPS)
- **Ticket Server** with multiple DB nodes, each owning a range of IDs
  ```
  Server A: IDs 1, 4, 7, 10 ...  (step=3, start=1)
  Server B: IDs 2, 5, 8, 11 ...  (step=3, start=2)
  Server C: IDs 3, 6, 9, 12 ...  (step=3, start=3)
  ```
- Or use **Twitter Snowflake** (64-bit IDs: timestamp + datacenter + sequence)

### Handling Custom Aliases
- Check availability in DB before assignment
- Store in same table with `custom_alias = TRUE`
- Risk: squatting. Mitigate with rate limiting + reserved word blocklist

---

## 🛡️ Edge Cases & Failure Scenarios

| Scenario | Solution |
|----------|----------|
| Same long URL submitted twice | Return existing short code (dedup by hashing long_url) |
| Short code not found | Return `404` with helpful message |
| Expired URL accessed | Return `410 Gone` |
| Malicious URL (phishing) | Integrate Google Safe Browsing API on write |
| Cache stampede on viral URL | Use **probabilistic early expiration** or **mutex lock** on cache miss |
| DB goes down | Serve from cache with stale-while-revalidate; circuit breaker pattern |

---

## 📊 Technology Choices Summary

| Layer | Choice | Reason |
|-------|--------|--------|
| **API** | Go / Node.js | High concurrency, low overhead |
| **Cache** | Redis Cluster | Sub-millisecond reads, TTL support |
| **Primary DB** | PostgreSQL | ACID, reliable, strong ecosystem |
| **Analytics DB** | ClickHouse | Columnar, fast aggregations |
| **Message Queue** | Kafka | High-throughput click event streaming |
| **ID Generation** | Snowflake IDs | Distributed, monotonic, no coordination |
| **CDN** | CloudFront / Fastly | Edge redirect for viral URLs |
| **Load Balancer** | AWS ALB | Layer 7, health checks, SSL termination |

---

## 🎙️ Interview Tips

> **Q: How do you handle the same long URL being submitted multiple times?**  
> A: Hash the long URL and check for an existing mapping in a secondary index. Return the existing short code instead of generating a new one. Trade-off: this requires an extra lookup on every write.

> **Q: What if the ID generator is a single point of failure?**  
> A: Use a multi-master ticket server setup, or move to Snowflake IDs which are generated locally per service instance (no central coordination).

> **Q: How would you make this geo-aware?**  
> A: Deploy regional clusters. Use GeoDNS to route users to the nearest cluster. Replicate the mapping table across regions with eventual consistency.

---

## 📚 Further Reading
- [How bit.ly handles 6 billion monthly clicks](https://bit.ly/engineering)  
- [Twitter Snowflake ID Generation](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake)  
- [Designing Data-Intensive Applications — Ch. 5 (Replication)](https://dataintensive.net/)

---

*Posted as part of the **Architecture of the Day** series. Next up: **Design a Distributed Cache (like Redis)***
