# Chapter 38 — Indexing

You've cut the catalogue to a few queries (Chapter 36) and bounded them to a page (Chapter 37). But a single, bounded query can *still* be slow — because of *how* the database finds the rows. Ask for published products sorted by date and, without help, Postgres reads **every row in the table**, checks each one, and sorts the survivors. That's a **sequential scan**, and on a million-row table it's slow no matter how few rows you ultimately return.

**Indexes** are the fix, and this chapter finally delivers on a promise made way back in Chapter 14: the optimization order is *normalize → measure → **index** → only then denormalize/cache*. You're at the index step. And crucially, you'll learn to **read the query plan** — to *see* the database scanning, add the right index, and *see* it seek instead. No guessing; the plan tells you.

## Where we're headed

By the end you can read an `EXPLAIN` plan, recognize a sequential scan, add indexes to the columns your catalogue filters and sorts on, and confirm via the plan and timing that the query now uses an index — turning scans into seeks.

## Step 1 — What an index is

An index is a separate, sorted data structure that lets the database **find rows without scanning the whole table** — exactly like the index at the back of a book lets you jump to a topic instead of reading every page. The default kind is a **B-tree**: a balanced tree that keeps a column's values in sorted order, so the database can binary-search to a value (or a range) in a handful of steps instead of reading millions of rows.

Without an index on `status`, "find published products" means *check every row's status*. With one, the database jumps straight to the published rows. The bigger the table, the more dramatic the difference — and a marketplace catalogue only grows.

## Step 2 — Read the query plan (measure, don't guess)

You never add indexes by hunch — you let the database show you what it's doing with **`EXPLAIN`** (the plan) and **`EXPLAIN ANALYZE`** (the plan *plus* actual timings). Run it on your catalogue query:

```sql
EXPLAIN ANALYZE
SELECT * FROM products WHERE status = 'published' ORDER BY created_at DESC, id DESC LIMIT 20;
```

Read the output for two words:

- **`Seq Scan`** — "sequential scan": the database read the *entire* table. On a large table, this is your slow query, found.
- **`Index Scan`** (or `Index Only Scan`) — it used an index to seek directly. This is the goal.

The plan also shows estimated and actual rows and time. Learning to glance at a plan and spot `Seq Scan` on a big table is one of the most useful database skills you'll build — it turns "the page feels slow" into "*here* is the scan, on *this* column."

## Step 3 — Index what you filter, sort, and join on

Indexes aren't free (Step 5), so you index *deliberately* — the columns that appear in `WHERE`, `ORDER BY`, and `JOIN`. For your app that means:

- **`products.status`** — every catalogue query filters by it.
- **`products(created_at, id)`** — your cursor sort (Chapter 37). A **composite index** matching the sort order lets the database both filter *and* return rows already sorted, making cursor pagination genuinely fast.
- **Foreign keys** — `products.store_id`, `orders.store_id`, `order_items.order_id`. These are joined and scoped on constantly (Chapters 28–29); Postgres does *not* index foreign keys automatically, and an unindexed FK makes joins and tenant-scoped queries scan.
- A **partial/composite index** matching your real query — e.g. `(status, created_at DESC, id DESC)` — can serve the whole catalogue query (filter + sort) from one index. Match the index to the query you actually run.

The principle: **index the columns the database has to search or sort by**, guided by the plans you read — not every column, and not by guesswork.

## Step 4 — JSONB and text need different indexes (a note)

Two cases your app will hit need non-default index types, worth flagging now:

- **JSONB attributes** (Chapter 9): querying inside the `attributes` document ("all mugs with a celadon glaze") wants a **GIN index** on the JSONB column — the index type built for "does this document contain this key/value." (The hint from Chapter 9 pointed here.)
- **Text search** (Chapter 39, next): searching product titles/descriptions wants a full-text (GIN) index, not a B-tree — because a B-tree can't help a `%term%` search. You'll set this up next chapter.

For now, know that "add an index" sometimes means "add the *right kind* of index for the query."

## Step 5 — The cost (why not index everything?)

Indexes speed reads but aren't free, so you don't blanket the table:

- **Writes get slower.** Every `INSERT`/`UPDATE`/`DELETE` must also update each index on that table. Over-indexing slows down exactly the writes a busy marketplace does constantly.
- **They take storage** and memory.

So the discipline is: index the columns your **measured, real queries** filter/sort/join on, verify each index is actually *used* (the plan shows an index scan), and drop indexes nothing uses. This is the same evidence-driven mindset as Chapter 14 — and note the payoff it promised: a good index often makes a query fast enough that you **never need to denormalize or cache** it. Index first; reach for the heavier tools only if indexing isn't enough.

## Step 6 — Do it on your project (hands-on)

**1. See the scan.** Run `EXPLAIN ANALYZE` on your catalogue query at volume and find the **`Seq Scan`**. Note the time.

**2. Add the indexes** — via a **new migration** (Chapter 12; never edit an applied one): `status`, the composite `(status, created_at DESC, id DESC)` for the catalogue+cursor, and your foreign keys (`store_id`, `order_id`). `npm run migrate`.

**3. See the seek.** Re-run `EXPLAIN ANALYZE`:

```
before:  Seq Scan on products  (reads all rows)        — slow, grows with table
after:   Index Scan using products_status_created_idx  — seeks directly, fast
```

Confirm the plan now says **Index Scan** and the time dropped. That before/after — scan to seek, in the plan and the timing — is the chapter's proof.

**4. Verify your indexes are used, not just created.** Check that each index you added actually appears in a plan for a real query; an index nothing uses is pure write-cost. Drop any that don't earn their place.

> 💡 **Hint — a composite index's column order matters.** An index on `(status, created_at, id)` helps queries that filter on `status` (the leftmost column) and then sort/range on the rest. It does *not* efficiently serve a query that filters only on `created_at` without `status`. Order the columns to match how your query filters then sorts — leftmost is what you filter by equality, then the range/sort columns. (Search "leftmost prefix rule" if the plan ignores your index.)

> **📖 Mandatory read — before Chapter 39.** Read **"Use the Index, Luke"** (a free, excellent SQL indexing resource — search that name) on B-trees and reading `EXPLAIN`. *Required: Chapter 39's search builds directly on indexing, with a different index type.*

> **Interesting to read.** A huge fraction of real "the database is slow" incidents are a single missing index on a column that a query filters on millions of rows — found in minutes by reading one `EXPLAIN` plan. Search *"missing index slow query EXPLAIN"* to see why "read the plan first" is the experienced engineer's reflex, not "add more servers."

## Definition of Done

Things you can **see or run** — and the gate to Chapter 39:

- [ ] You ran **`EXPLAIN ANALYZE`** on the catalogue query and identified a **`Seq Scan`** at volume
- [ ] You added indexes via a **new migration** for `status`, the catalogue/cursor **composite**, and **foreign keys**
- [ ] Re-running `EXPLAIN ANALYZE` shows an **`Index Scan`** and a **lower time** — you recorded before/after
- [ ] You **verified each index is actually used** by a real query (and would drop unused ones)
- [ ] You can explain the **write/storage cost** of indexes and why you don't index everything
- [ ] You can state why indexing comes **before** denormalizing/caching (Chapter 14's order)

> **✍️ Log it (mandatory).** In `learning-log/38-indexing.md` — **decision** first, then **topics**: **(Decision)** How did you decide *which* columns to index, and why index deliberately rather than indexing everything? Why does indexing come before caching/denormalizing in the optimization order? **(Topics)** (1) What is a B-tree index and how does it turn a scan into a seek? (2) How do you read `EXPLAIN` to tell a `Seq Scan` from an `Index Scan`? (3) Why does composite-index column order matter for your catalogue query? (4) What does an index cost, and how do you know one is worth keeping?

*All boxes ticked and the log written? Then continue. The catalogue is fast to filter and sort — but customers don't browse, they *search*. And search needs a different kind of index entirely.*

---

Next: customers want to *find* things — add search and filtering, and meet the index type built for text. → **[Chapter 39 — Search and filtering](39-search-and-filtering.md)**
