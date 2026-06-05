# Chapter 39 — Search and filtering

Your catalogue is fast to list, page, and sort (Chapters 35–38). But customers don't browse a marketplace page by page — they **search**: "ceramic mug," "merino," "under $30." Search and filtering are how a shopper actually finds anything, and they introduce a problem your B-tree indexes can't solve: matching *words inside text*. This chapter adds real search and structured filtering, and in doing so meets the index type built for language — closing out the performance module.

It pulls together everything from the last four chapters: search results are **filtered** (visibility rules), **paginated** (cursor), and **indexed** (the right kind) — search is where all of Week 3 converges.

## Where we're headed

By the end customers can search products by keyword (using proper full-text search, not slow `LIKE`), filter by structured fields (price range, attributes), and get results that are still published-only, paginated, and index-backed.

## Step 1 — Filtering vs searching: two different jobs

They sound similar but are different operations:

- **Filtering** narrows by *structured, exact* criteria: price between X and Y, a specific attribute, in stock. These map to `WHERE` clauses on columns or JSONB, and they ride on the B-tree/GIN indexes from Chapter 38.
- **Searching** matches *free text* against titles and descriptions: "hand thrown bowl." This is fuzzy, language-aware matching — and it needs a different tool.

A real catalogue does both at once ("merino scarves under $50"), so you'll combine them. But treat search as its own beast, because the naive approach to it is a performance trap.

## Step 2 — The wrong way to search: `LIKE '%term%'`

The obvious way to search titles is a wildcard match:

```sql
-- ❌ tempting, and slow + dumb
SELECT * FROM products WHERE title ILIKE '%mug%';
```

Two problems, one performance and one quality. **Performance:** a leading-wildcard `%mug%` **cannot use a B-tree index** (the index is sorted by the *start* of the string; a leading `%` defeats it), so this is a full sequential scan every search — exactly the `Seq Scan` you just learned to eliminate. **Quality:** it's a dumb substring match. It won't find "mugs" when you search "mug" intelligently, ignores word stems, can't rank by relevance, and matches "smuggler" for "mug." `LIKE` is fine for the occasional admin lookup; it is not search.

## Step 3 — The right way: full-text search

Postgres has **full-text search** built in, designed for exactly this. Instead of substring matching, it understands *words*: it breaks text into normalized tokens (**lexemes** — "running" and "ran" both reduce to "run"), ignores noise words, and ranks results by relevance. The pieces:

- A **`tsvector`** — the searchable, tokenized form of a product's text (title + description).
- A **`tsquery`** — the parsed search terms.
- A **GIN index** on the `tsvector` — the text-search index type (Chapter 38 flagged this), which makes the match fast instead of a scan.

```sql
-- ✅ real search: tokenized, ranked, index-backed
SELECT * FROM products
WHERE to_tsvector('english', title || ' ' || description) @@ websearch_to_tsquery('english', 'ceramic mug')
  AND status = 'published'
ORDER BY ts_rank(...) DESC;
-- with a GIN index on the tsvector, this is fast, not a scan
```

This is the standard, no-extra-infrastructure choice for a project this size — you get language-aware, ranked, indexed search from the database you already run. (At much larger scale, teams add a dedicated engine like Elasticsearch/OpenSearch; that's beyond this course's scope, and Postgres full-text search is the right tool here. Note that trade-off rather than reaching for a search cluster prematurely.)

## Step 4 — Filtering on structured fields and JSONB

Alongside text search, support structured filters, all validated (Chapter 16) and composed onto the published-only scope:

- **Price range** — `price_cents BETWEEN $min AND $max`, on an indexed column.
- **Attributes** (Chapter 9's JSONB) — "celadon glaze," "merino fibre" — queried with JSONB operators and served by the **GIN index** on `attributes` (Chapter 38). This is where the hybrid model pays off: variable attributes are filterable without schema changes.
- All filters **compose**: search terms + price range + attribute + `status='published'`, then cursor-paginated (Chapter 37).

Validate every filter input — ranges are numbers, attribute keys come from an allowlist where it matters (the Chapter 33 sorting lesson applies to filters too).

## Step 5 — Do it on your project (hands-on)

**1. Add full-text search** to the catalogue: build a `tsvector` over title + description, add a **GIN index** for it (new migration, Chapter 12), and accept a `q` search param that uses `websearch_to_tsquery`, ranking by relevance.

**2. Add structured filters** — `minPrice`/`maxPrice` and at least one attribute filter — validated and composed onto the published scope and the cursor pagination.

**3. Prove it works and is fast:**

```bash
# keyword search, ranked
curl -s "localhost:3000/products?q=ceramic%20mug&limit=20" | jq '.items[].title'

# search + price filter + pagination, combined
curl -s "localhost:3000/products?q=mug&maxPrice=3000&limit=20" | jq '{count:(.items|length), next:.nextCursor}'
```

**4. Confirm it uses the index, not a scan.** `EXPLAIN ANALYZE` the search query and verify a **GIN/Index Scan**, not a `Seq Scan` — the Chapter 38 reflex applied to search. Compare against a `LIKE '%mug%'` plan to *see* the difference.

```sql
EXPLAIN ANALYZE SELECT ... @@ websearch_to_tsquery('english','mug') ...;   -- index scan, fast
EXPLAIN ANALYZE SELECT ... WHERE title ILIKE '%mug%';                      -- seq scan, slow
```

Watching the full-text query seek while the `LIKE` query scans is the proof that you chose the right tool.

> 💡 **Hint — keep the `tsvector` in sync, or precompute it.** Computing `to_tsvector(...)` on every query works but recomputes each time. For real use, store the `tsvector` in a (generated/maintained) column and index *that*, so the tokenization happens on write, not on every search. It's the same "do the work once, not per request" instinct as fixing N+1 (Chapter 36) — just applied to tokenization.

> **📖 Mandatory read — before Chapter 40.** Read the **PostgreSQL full-text search** intro (search *"postgres full text search tutorial"*) and a short piece on **when to use Postgres FTS vs a dedicated search engine**. *Required: Chapter 40 begins caching, and the catalogue search is a prime caching target.*

> **Interesting to read.** Postgres full-text search is surprisingly capable — many production apps serve millions of searches on it before ever needing Elasticsearch, and "we removed our search cluster and just used Postgres" is a common engineering story. Search *"postgres full text search vs elasticsearch"* to see where the line actually sits — and why reaching for a search cluster too early is a classic over-engineering trap.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 40:

- [ ] The catalogue supports **keyword search** via **Postgres full-text search** (tokenized, ranked), not `LIKE '%term%'`
- [ ] A **GIN index** backs the search; `EXPLAIN ANALYZE` shows an index scan, and you compared it to the `LIKE` scan
- [ ] **Structured filters** (price range, at least one attribute) work, are **validated**, and **compose** with search
- [ ] Search/filter results remain **published-only** and **cursor-paginated**
- [ ] You can explain why `LIKE '%term%'` is both slow and poor quality, and why FTS fixes both
- [ ] You can state when Postgres FTS suffices and when a dedicated engine would be warranted

> **✍️ Log it (mandatory).** In `learning-log/39-search-and-filtering.md` — **decision** first, then **topics**: **(Decision)** Why use Postgres full-text search instead of `LIKE '%term%'`, and why *not* reach for a dedicated search engine at this stage? **(Topics)** (1) What's the difference between filtering and searching, and which index type serves each? (2) Why can't a B-tree index help a leading-wildcard `LIKE`? (3) How does full-text search differ from substring matching (lexemes, ranking)? (4) How do search, filters, the published scope, and pagination compose into one query?

*All boxes ticked and the log written? Then continue. Your catalogue is fast and searchable — but the most popular searches re-run the same work over and over. Next you stop recomputing what hasn't changed.*

---

Next: even a fast query is wasteful if you run it thousands of times for the same answer — start caching the hot reads. → **[Chapter 40 — Caching](40-caching.md)**
