# Chapter 41 — Cache invalidation

Your cache makes the catalogue instant (Chapter 40) — but you watched it lie: edit a product, and the cache keeps serving the old version until the TTL expires. There's a famous line in computer science — *"there are only two hard things: cache invalidation and naming things"* — and you've just met the first one. A stale cache is worse than no cache, because it confidently serves wrong answers. This chapter makes the cache update the moment the underlying data changes, so you get speed *and* freshness.

The hard part isn't the technique — it's the *discipline*: every write path that affects cached data must invalidate it, and forgetting *one* leaves users staring at stale data. It's the Chapter 14 warning made literal — a stale copy is a bug — now with a Redis key instead of a denormalized column.

## Where we're headed

By the end, editing or publishing a product **immediately** refreshes what the catalogue serves (no waiting for TTL), via explicit invalidation on every write path, with TTL kept as a backstop.

## Step 1 — The strategies

There are three ways to keep a cache from going stale, and they combine:

| Strategy | How | Trade-off |
|---|---|---|
| **TTL only** (Chapter 40) | Let entries expire after N seconds | Simple, but stale until expiry |
| **Explicit invalidation** | On a write, delete/refresh the affected cache entries | Fresh immediately — *if* you catch every write path |
| **Write-through** | Update the cache as part of the write itself | Always consistent, more coupling |

**The course uses explicit invalidation (delete-on-write) with TTL as a backstop.** When a vendor edits, publishes, or deletes a product, you *delete the affected cache keys* so the next read misses and re-fetches fresh data. TTL stays on as a safety net for any invalidation you miss or any derived cache you forgot — belt and braces.

## Step 2 — The discipline: every write must invalidate

Here's the trap, stated plainly. Your cached catalogue depends on product data. *Every* operation that changes product data — `PATCH /products/:id`, publish/unpublish, `DELETE`, an admin change, a future bulk import — must invalidate the relevant cache. Miss one path, and that path silently leaves stale data behind until TTL.

So invalidation can't be an afterthought sprinkled per-handler (the Chapter 16 rot again). Centralize it: a small **invalidation step tied to product writes**, so "this changed → drop these keys" happens consistently wherever products change. The principle mirrors your scoping helper (Chapter 28): make the correct thing the default path, not something each handler remembers.

## Step 3 — What to invalidate (the cascade)

A single product edit can affect *several* cached entries, and you must drop all of them:

- The cached **catalogue pages** that might include this product.
- A cached **single product** entry, if you cache those.
- Cached **search results** that might match it.

This is where caching gets genuinely hard — knowing *which* keys a change affects. Two common tactics: **key by dependency** (so you can find all keys touching a product) or **coarse invalidation** (drop the whole catalogue cache namespace on any product change — simpler, slightly less efficient, often the right call early). Choose based on how granular you need to be; for a project this size, coarse invalidation of the catalogue on any product write is a sound, simple default.

## Step 4 — Do it on your project (hands-on)

**1. Add invalidation to product writes.** On `PATCH /products/:id`, publish/unpublish, and `DELETE` (Chapter 32), delete the affected catalogue (and product/search) cache keys after the write commits.

**2. Centralize it** so every product-write path runs the same invalidation — not copy-pasted per handler.

**3. Prove fresh-and-fast:**

```bash
# warm the cache
curl -s "localhost:3000/products?limit=20" | jq '.items[0]'        # cached now

# edit that product
curl -s -X PATCH localhost:3000/products/$PID -H "authorization: Bearer $V" -d '{"title":"New name"}' -H 'content-type: application/json'

# reload immediately → the change is visible NOW, not after TTL
curl -s "localhost:3000/products?limit=20" | jq '.items[0]'        # → updated title, fresh
# and confirm the cache re-warmed (a second read is a hit again)
```

Editing a product and seeing the catalogue reflect it *immediately* — while still serving subsequent reads from cache — is invalidation working: fresh *and* fast.

**4. Test it (regression guard).** Add a test: cache a read, perform a write, assert the next read reflects the change. Like your isolation test (Chapter 30), this guards against someone adding a new write path later that forgets to invalidate.

> 💡 **Hint — invalidate *after* the write commits, and beware stampedes.** Drop the cache key only once the database write has succeeded (and committed — relevant once checkout transactions arrive in Chapter 47); invalidating before risks a concurrent read re-caching the *old* value. And when a hot key is invalidated, many requests can miss at once and all hammer the database — a **cache stampede**. For a popular catalogue, know this exists (mitigations: a short lock, or serve-stale-while-revalidate); it's a real concern at scale.

> **📖 Mandatory read — before Chapter 42.** Read about **cache invalidation strategies** and **cache stampede / thundering herd** (search those terms). *Required: invalidation correctness is the difference between a cache that helps and one that lies.*

> **Interesting to read.** "Cache invalidation is one of the two hard problems" is a joke precisely because it's true — real outages and "why is the old price showing?" bugs trace to a missed invalidation path. Search *"hardest problem cache invalidation"* to appreciate why this chapter is short but the bugs are long-lived.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 42:

- [ ] Product **writes invalidate** the affected cache entries, centrally (not per-handler ad hoc)
- [ ] Editing/publishing a product is reflected in the catalogue **immediately**, not after TTL (you verified)
- [ ] **TTL remains** as a backstop for anything invalidation misses
- [ ] Invalidation runs **after** the write succeeds
- [ ] A **test** asserts a write is reflected in the next cached read
- [ ] You can explain what a **cache stampede** is and why it matters for a hot key

> **✍️ Log it (mandatory).** In `learning-log/41-cache-invalidation.md` — **decision** first, then **topics**: **(Decision)** Why explicit invalidation *plus* TTL rather than TTL alone, and what discipline does explicit invalidation demand? Why centralize invalidation rather than add it per handler? **(Topics)** (1) Which write paths must invalidate the catalogue cache, and what happens if one is missed? (2) Why invalidate *after* the write commits? (3) What is a cache stampede? (4) How is a stale cache the same failure as a stale denormalized copy (Chapter 14)?

*All boxes ticked and the log written? Then continue. Your data layer is fast and correct — but handmade goods are sold on their *photos*, and you're not storing images yet. Time to handle files properly.*

---

Next: a marketplace for handmade goods lives on its product photos — decide how to store and serve images at scale. → **[Chapter 42 — Images and object storage: the concept](42-images-concept.md)**
