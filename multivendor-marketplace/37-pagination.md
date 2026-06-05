# Chapter 37 — Pagination

You fixed the catalogue's query *count* (Chapter 36) — but it still does something reckless: it returns **every published product in the marketplace** in one response. Fine with seed data; catastrophic at scale. Ten thousand products is a multi-megabyte payload, a slow query, and a browser asked to render a list nobody will scroll. No real catalogue loads everything — it loads a **page** at a time. This chapter adds pagination, and it carries a genuine design decision with a clear winner for a catalogue.

It also cashes in a promise from Chapter 16: the validated pagination query schema you were told you'd reuse is now the thing you build against.

## Where we're headed

By the end the catalogue returns a bounded page of results with a way to fetch the next, using **cursor (keyset) pagination** with a validated page size — chosen deliberately over offset pagination, and tested at volume.

## Step 1 — The two approaches

There are two ways to paginate, and the difference matters more than it first appears.

**Offset pagination** — "skip 40, take 20": `LIMIT 20 OFFSET 40`. Page 3 of 20-per-page. Familiar and simple.

**Cursor (keyset) pagination** — "give me 20 *after* this item": `WHERE (created_at, id) < (<last seen>) ORDER BY created_at DESC, id DESC LIMIT 20`. The client passes back a **cursor** (an opaque pointer to the last item it saw) to get the next page.

```
offset:  GET /products?limit=20&offset=40      → rows 41–60
cursor:  GET /products?limit=20&cursor=<opaque> → 20 rows after the cursor
```

## Step 2 — Why offset breaks down (the contrast)

Offset is fine for small, static data and tempting because it's simple — but for a large, changing catalogue it has two real problems:

- **It gets slower the deeper you go.** `OFFSET 100000` forces the database to *find and discard* 100,000 rows before returning 20. The work grows with the page number — deep pages crawl. (You'll feel this in Step 4.)
- **It skips and duplicates rows when data changes.** If a new product is inserted while a user pages, every subsequent offset shifts by one — they see an item twice, or miss one entirely. For a live catalogue where products are constantly added, this is a real, visible glitch.

**Cursor pagination** has neither problem: it asks for rows *after a stable key*, so the database jumps straight there using an index (no counting-and-discarding), and inserting a row elsewhere doesn't shift the cursor's position. Its trade-off is that you **can't jump to "page 7"** — only "next/previous" — which for an infinite-scroll catalogue is exactly the interaction you want anyway.

| | Offset | Cursor (keyset) |
|---|---|---|
| Deep-page speed | Degrades (skips N rows) | Constant (seeks via index) |
| Stable under inserts | No (skips/dupes) | Yes |
| Jump to arbitrary page | Yes | No (next/prev only) |
| Best for | Small/static, admin tables | Large, live feeds & catalogues |

**The course uses cursor pagination for the catalogue** — it scales and stays correct under constant change, and "next page / infinite scroll" is the right UX for browsing. (Offset is a fine choice for small bounded admin lists; pick per use case, but the catalogue is cursor.)

## Step 3 — The pieces of a correct cursor

A cursor needs a **stable, unique sort order**. Sorting by `created_at` alone isn't enough — two products can share a timestamp, causing rows to be skipped at page boundaries. So sort by a **tiebreaker pair**: `(created_at, id)` — the unique `id` (Chapter 8) guarantees a total order. The cursor encodes the last row's `(created_at, id)`; the next query asks for rows strictly after it.

And **always bound the page size**: validate `limit` (Chapter 16) to a sane maximum (say, 50). Otherwise a client requests `limit=1000000` and you're back to returning everything. The response returns the page plus the **next cursor** (or null at the end):

```
GET /products?limit=20&cursor=<opaque>
  validate(query): { limit: int 1–50 (default 20), cursor?: opaque string }
  → { items: [...20], nextCursor: "<opaque>" | null }
```

## Step 4 — Do it on your project (hands-on)

**1. Add cursor pagination** to `GET /products`: sort by `(created_at, id)`, accept a validated `limit` and optional `cursor`, return `items` + `nextCursor`.

**2. Validate the inputs** (Chapter 16): `limit` clamped to a max, `cursor` an opaque token you decode — reject a malformed cursor with `400`.

**3. Page through and feel the difference:**

```bash
# first page
curl -s "localhost:3000/products?limit=20" | jq '{count: (.items|length), next: .nextCursor}'
# next page using the returned cursor
curl -s "localhost:3000/products?limit=20&cursor=<paste nextCursor>" | jq '{count: (.items|length), next: .nextCursor}'
```

**4. Prove offset's deep-page cost (the measurement).** With your seeded volume, time an `OFFSET 10000 LIMIT 20` query versus the cursor query for a deep page:

```sql
EXPLAIN ANALYZE SELECT * FROM products WHERE status='published' ORDER BY created_at DESC, id DESC OFFSET 10000 LIMIT 20;
-- note the time — it scanned past 10,000 rows
```

The offset query does measurably more work for a deep page than the cursor query, which seeks straight in. That measured difference is *why* you chose cursor. (Chapter 38 makes the cursor query even faster with the right index.)

> 💡 **Hint — the cursor must match the sort.** A cursor only works if it encodes the *exact* columns you `ORDER BY`, in the same order, with a unique tiebreaker. If pages skip or repeat items, your sort and your cursor have drifted apart — they're a matched pair. And keep the cursor *opaque* (e.g. base64 the `(created_at, id)`), so clients treat it as a token, not something to construct or tamper with.

> **📖 Mandatory read — before Chapter 38.** Read a clear explainer on **cursor vs offset pagination** (search *"cursor pagination vs offset"* — the Slack and Shopify engineering posts are excellent). *Required: the trade-offs here recur for every large list, and Chapter 38 indexes the column your cursor sorts on.*

> **Interesting to read.** Large platforms moved to cursor pagination specifically because offset broke at their scale — deep pages timed out and live feeds skipped items. Search *"why we switched to cursor pagination"* to read real engineering accounts of the exact problems you just reasoned through.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 38:

- [ ] `GET /products` returns a **bounded page** with a **`nextCursor`**, not the whole catalogue
- [ ] It uses **cursor (keyset)** pagination over a **stable `(created_at, id)`** sort
- [ ] `limit` is **validated and capped**; a malformed cursor returns **`400`**
- [ ] You **paged through** multiple pages using the returned cursor
- [ ] You **measured** offset's deep-page cost versus cursor and can explain why cursor scales
- [ ] You can state offset's two failure modes (deep-page cost, skips/dupes under inserts)

> **✍️ Log it (mandatory).** In `learning-log/37-pagination.md` — **decision** first, then **topics**: **(Decision)** Why cursor pagination over offset for the catalogue, and what does cursor give up in exchange? Why cap the page size? **(Topics)** (1) Explain offset's two failure modes with your measured deep-page result. (2) Why must the cursor sort include a unique tiebreaker like `id`? (3) Why must the cursor encode exactly the `ORDER BY` columns? (4) When *would* offset pagination be the right choice?

*All boxes ticked and the log written? Then continue. Your queries are few and bounded — but are they *fast*? Time to look at how the database actually finds rows, and make it find them quickly.*

---

Next: a bounded query can still be slow if the database scans every row to satisfy it — learn to read the query plan and add the indexes that make it fast. → **[Chapter 38 — Indexing](38-indexing.md)**
