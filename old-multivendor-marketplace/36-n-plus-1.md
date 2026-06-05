# Chapter 36 — The N+1 query problem

Your catalogue works (Chapter 35), and on seed data it feels instant. Then you load it with the thousands of products you seeded, watch your database's query log, and find something alarming: rendering one catalogue page fired **hundreds of queries**. This is the **N+1 query problem** — the single most common performance bug in web development, and the canonical lesson this whole course has been building toward measuring rather than guessing.

This chapter is where the Chapter 14 discipline pays off: you don't *speculate* that something is slow, you **measure** it — see the exact query count — then fix it and measure again to prove the fix. Teaching by the concrete number (the query count) is the whole point.

## Where we're headed

By the end you can reproduce the N+1 on your catalogue (and *count* the queries), explain why it happens, fix it with a single joined or batched query, and measure the dramatic drop — turning hundreds of queries into one or two.

## Step 1 — See the problem (measure first)

The catalogue shows each product with its store name. The naive way to get that store name — and the way many ORMs do it *by default* if you're not careful — is: run one query for the products, then, **for each product**, run another query to fetch its store.

```
1 query:    SELECT * FROM products WHERE status = 'published' LIMIT 100;   -- 100 products
then for EACH of the 100:
  SELECT * FROM stores WHERE id = $1;   -- ...100 more queries
─────────────────────────────────────────────────────────────────────────
total: 1 + 100 = 101 queries to render ONE page
```

That's **N+1**: **1** query for the list, plus **N** more (one per row) for the related data. With 100 products it's 101 queries; with 1,000 it's 1,001. Each is a round-trip to the database, and they add up to a page that's fine in development and crawls in production.

**Don't take this on faith — see it.** Turn on your database/ORM query logging and load the catalogue. Count the queries. Watching `SELECT * FROM stores WHERE id = ...` scroll past a hundred times is the moment the abstract bug becomes concrete — and it's exactly the "measure the cost" teaching from Chapter 14.

## Step 2 — Why it happens

N+1 is rarely something you write *on purpose* — it hides. You loop over products to build the response, and somewhere in that loop you access `product.store.name`. If the store wasn't already loaded, *that property access fires a query*, once per iteration. ORMs make this especially easy to do invisibly with **lazy loading**: relations look like ordinary fields, but touching one triggers a hidden query. The loop you wrote looks innocent; the queries are emitted behind your back.

The root cause is structural: you asked for the related data **one row at a time** instead of **all at once**. The database can fetch a hundred stores in a single query as easily as one — you just have to ask it to.

## Step 3 — The fix: ask once, not N times

Two standard fixes, both turning N+1 into a small constant:

- **A join** — fetch products and their stores together in one query (you set this up in Chapter 35):

```sql
-- ✅ ONE query for products AND their store data
SELECT p.*, s.name AS store_name, s.slug AS store_slug
FROM products p JOIN stores s ON s.id = p.store_id
WHERE p.status = 'published' LIMIT 100;
```

- **Batch loading (eager loading)** — fetch the products, collect their `store_id`s, then fetch *all* the needed stores in **one** query (`WHERE id IN (...)`) and stitch them together in code. Most ORMs offer this as an "include"/"eager load"/"with" option that turns N+1 into 2 queries.

```
1 query: products
1 query: SELECT * FROM stores WHERE id IN (<the 100 store ids>)
total: 2 queries, regardless of page size
```

Either way, the page that took 101 queries now takes 1–2. **The fix isn't a faster query — it's far *fewer* queries.** Whether you join or batch is a minor choice (joins for display data, batching when relations are complex); eliminating the per-row query is the point.

## Step 4 — Do it on your project (hands-on)

**1. Reproduce and count.** With query logging on, load your catalogue at volume and **count the queries**. Confirm it's ~`N+1` (one per product for the store). Write the number down.

**2. Fix it.** Change the catalogue to load stores in **one** query — a join (Chapter 35's query) or your ORM's eager-load/include for the store relation.

**3. Measure again and compare:**

```
before:  load catalogue → 101 queries   (1 products + 100 stores)
after:   load catalogue →   1 query     (join)   — or 2 (batched)
```

Seeing the count collapse from 101 to 1 is the proof. Note the before/after in your log — the *number* is the evidence, exactly as Chapter 14 framed it.

**4. Guard against regressions.** N+1 loves to creep back when someone adds a new relation to the response. If your stack can **assert a query count** in a test (some can fail a test if a request exceeds N queries), add one for the catalogue — the same regression-guard instinct as your isolation test (Chapter 30).

> 💡 **Hint — N+1 hides anywhere you loop over rows and touch a relation.** It's not just the catalogue: an order list that shows each order's items, a vendor list showing each store's product count — any "list of things, each needing related data" is a candidate. The tell is a query *inside a loop*. Whenever you build a list endpoint, ask: "am I fetching the related data once, or once per row?"

> **📖 Mandatory read — before Chapter 37.** Read about your ORM's **eager loading / N+1 prevention** (search *"<your ORM> N+1 eager loading"*) and a general explainer on the **N+1 problem**. *Required: every list endpoint you build from here needs this reflex.*

> **Interesting to read.** N+1 is so pervasive that entire tools exist just to detect it (the "Bullet" gem, ORM query-count assertions, APM flame graphs) — and it's a top cause of "the app got slow as we grew" incidents, because it's *invisible* until there's volume. Search *"N+1 query problem"* to see why it's the first thing experienced engineers check on a slow list page.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 37:

- [ ] You **reproduced the N+1** on the catalogue at volume and **counted** the queries (~N+1)
- [ ] You **fixed it** with a single joined or batched query, so the page is now **1–2 queries regardless of size**
- [ ] You **measured the before/after** query counts and recorded the drop
- [ ] You can explain *why* lazy-loading a relation in a loop causes N+1
- [ ] (If supported) a test **asserts the catalogue's query count** stays low, guarding the regression

> **✍️ Log it (mandatory).** In `learning-log/36-n-plus-1.md` — **decision** first, then **topics**: **(Decision)** Why fix N+1 by reducing the *number* of queries rather than by speeding up each query? Why measure the query count before and after instead of trusting that the fix worked? **(Topics)** (1) Explain the N+1 problem with your catalogue's exact before/after numbers. (2) Why does lazy loading make N+1 easy to introduce invisibly? (3) What's the difference between fixing it with a join versus eager/batch loading? (4) Where *else* in your app could N+1 hide, and what's the tell?

*All boxes ticked and the log written? Then continue. One query now, but it still returns *every* published product. Next you stop loading the whole catalogue at once.*

---

Next: one query is good, but returning ten thousand products in it is not — paginate the catalogue so it scales. → **[Chapter 37 — Pagination](37-pagination.md)**
