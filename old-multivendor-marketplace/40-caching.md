# Chapter 40 — Caching

Your catalogue is now fast to query — few queries (Chapter 36), bounded pages (Chapter 37), index-backed (Chapter 38), searchable (Chapter 39). But step back and notice the waste: the *same* popular searches and the *same* front-page catalogue run the *same* query thousands of times an hour, recomputing an answer that hasn't changed. Even a fast query is wasteful when you run it ten thousand times for an identical result. **Caching** stops that — store the computed answer somewhere fast and serve it again without touching the database.

This is the last rung of the Chapter 14 optimization ladder — *normalize → measure → index → **then** cache* — and you reach it deliberately, after indexing, not before. Caching is powerful and dangerous in equal measure: it makes reads instant, but it introduces the possibility of serving *stale* data, which is the entire subject of the next chapter.

## Where we're headed

By the end you cache hot catalogue reads in Redis using the cache-aside pattern with a TTL, you measure the hit-vs-miss difference, and you understand the staleness trade-off that Chapter 41 resolves.

## Step 1 — What to cache (and what not to)

Caching trades freshness and memory for speed, so it earns its place only on the right data:

- **Cache:** reads that are **frequent**, **expensive** to compute, and **change rarely** relative to how often they're read — the public catalogue, a popular product page, search results for common terms.
- **Don't cache:** data that's cheap to fetch, rarely read, or must always be perfectly current (a vendor's live order status, a customer's cart). Caching these adds staleness risk for little gain.

The catalogue is the textbook candidate: read constantly by everyone, relatively expensive (join + filter + sort), and changing only when a vendor edits a product.

## Step 2 — The cache-aside pattern

The standard pattern is **cache-aside** (lazy caching), and the flow is simple:

```
read request →
  1. look in the cache (Redis) for this key
  2. HIT  → return the cached value (no database touched)
  3. MISS → query the database, store the result in the cache (with a TTL), return it
```

The first request for something is a miss (pays full cost, warms the cache); every subsequent request within the TTL is a hit (instant). You already have Redis running (Chapter 5, via Docker) and a `redis` client wired through config (Chapter 6) — it does double duty as your cache here and your job queue later (Week 4).

## Step 3 — Keys and TTL

Two things make a cache correct:

- **The key** uniquely identifies the cached answer. The catalogue's first page is one key; a search for "mug" page 2 is another. Build keys from the query that produced the result — `catalogue:published:page1`, `search:mug:cursor:<x>` — so different queries don't collide on one entry.
- **The TTL (time to live)** is how long the entry stays valid before Redis evicts it. It's your *crude* staleness bound: a 60-second TTL means cached data can be at most 60 seconds out of date. A short TTL means fresher data but more misses; a long TTL means faster but staler. TTL alone is the simplest invalidation — it's "let it go stale, then expire" — and Chapter 41 adds precise invalidation on top.

## Step 4 — The trade-off you're accepting (the cliffhanger)

Caching introduces one hard problem: **the cache can be wrong.** The moment you store a copy of the catalogue, a vendor can edit a product and your cache keeps serving the *old* version until the TTL expires. This is denormalization-for-performance (Chapter 14) wearing a timer — a deliberate, temporary copy whose divergence from the source is a *bug* you manage, not a feature. TTL bounds the staleness crudely; making the cache update *immediately* on a write is **cache invalidation**, the next chapter (and famously one of the two hard problems in computer science). For now, accept TTL-bounded staleness and feel where it's not good enough.

## Step 5 — Do it on your project (hands-on)

**1. Cache the catalogue read** with cache-aside: on `GET /products`, build a cache key from the query, check Redis, return on hit; on miss, query the DB, store in Redis with a TTL (e.g. 60s), return.

**2. Measure hit vs miss:**

```bash
# first request → MISS (queries DB, warms cache) — note the time
time curl -s "localhost:3000/products?limit=20" > /dev/null
# second request → HIT (served from Redis) — note the much lower time
time curl -s "localhost:3000/products?limit=20" > /dev/null
# inspect the cached key
redis-cli KEYS 'catalogue:*'
```

The second request being dramatically faster (and firing *no* database query — check your query log, Chapter 36) is the cache working.

**3. Feel the staleness.** Edit a product (Chapter 32) and immediately reload the catalogue: within the TTL, you still see the **old** data. That's the problem — and exactly what Chapter 41 fixes. Note it.

> 💡 **Hint — cache the result, not the trip.** Cache the *computed answer* (the shaped catalogue page), not just "the DB rows" — so a hit skips the query *and* the response-shaping work. And always set a TTL on every cache entry: a cache without expiry is a memory leak and an unbounded-staleness bug waiting to happen. Every key gets a TTL, even when you also invalidate explicitly (Chapter 41) — TTL is your safety net for the invalidation you forget.

> **📖 Mandatory read — before Chapter 41.** Read about the **cache-aside pattern** and **Redis TTL/expiry** (search *"cache aside pattern"*). *Required: Chapter 41 builds invalidation directly on this.*

> **Interesting to read.** Caching is how read-heavy sites survive traffic that would melt their databases — the front page of a huge marketplace might serve from cache 99% of the time, hitting the database only on misses. Search *"cache hit ratio"* to see why a few percentage points of hit rate is worth serious engineering.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 41:

- [ ] The catalogue read uses **cache-aside** with **Redis**: hit serves from cache, miss queries the DB and warms it
- [ ] Every cache entry has a **TTL**; cache **keys** distinguish different queries/pages
- [ ] You **measured** a hit being dramatically faster than a miss and confirmed a hit fires **no DB query**
- [ ] You **observed stale data** after editing a product within the TTL (the problem Chapter 41 solves)
- [ ] You can explain *what* to cache and what not to, and why caching comes **after** indexing

> **✍️ Log it (mandatory).** In `learning-log/40-caching.md` — **decision** first, then **topics**: **(Decision)** Why cache the catalogue but not a customer's cart or live order status? Why does caching come *after* indexing in the optimization order (Chapter 14)? **(Topics)** (1) Walk through the cache-aside flow on a hit and a miss. (2) What makes a good cache key, and what does TTL bound? (3) How is caching a form of denormalization-for-performance? (4) Describe the staleness you observed and why TTL alone isn't enough.

*All boxes ticked and the log written? Then continue. Your cache is fast but can serve old data — next you make it update the instant something changes.*

---

Next: a fast cache that serves stale data is a bug — invalidate it precisely so reads are fresh *and* fast. → **[Chapter 41 — Cache invalidation](41-cache-invalidation.md)**
