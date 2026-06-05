# Chapter 14 — Normalization vs denormalization

Step back and look at what you built this module. You split everything into clean, separate tables — accounts here, stores there, products somewhere else — each fact living in exactly one place, stitched together with foreign keys. That's **normalization**, and it was the right instinct. And yet, in Chapter 10, you did the *opposite* on purpose: you **copied** a product's price and title onto the order line, accepting two copies of the same fact. Were those two chapters contradicting each other?

No — and understanding *why not* is the single idea that governs every data-modelling decision you'll make for the rest of the course. This is a concept chapter: there's nothing new to build. Instead you'll name the principle behind choices you already made by instinct, learn the framework for the ones ahead, and audit your own schema against it.

## What you'll be able to explain

By the end you can say, plainly, what normalization and denormalization are, the *two different reasons* you'd ever copy data, why one of them is risky and the other isn't, and the rule for deciding — so you never denormalize out of vague fear that a join is "probably slow."

## Section 1 — Normalization: every fact in one place

**Normalization** is organising data so each fact is stored *once*. A vendor's display name lives in that vendor's row and nowhere else. A product's price lives on the product. Nothing is copied; everything is referenced. Your entire schema is built this way.

The point of it is to make a whole category of bug *impossible*: the **update anomaly**. If a fact lives in one place, changing it is one update and the system is instantly consistent. There's no way for two copies to disagree, because there are no copies.

You don't need the academic theory, but the practical heart of the "normal forms" is worth holding in plain words:

- Don't cram many values into one field or repeat columns (no `tag1, tag2, tag3`, no array of line items stuffed onto the order row — which is exactly why `order_items` is its own table).
- Every column should describe *the thing the row is about*, depending only on its key. A product's columns describe the product; the vendor's *name* describes the vendor, so it belongs on the vendor — **not copied onto every product.**

That last clause is the one that matters, and it's the line denormalization deliberately crosses.

## Section 2 — The cost of normalization: joins

Normalization isn't free. Because the data is split across tables, assembling a *whole picture* means **joining** them back together. To render one product page you might join `products` → `stores` → `accounts` to show "Celadon Mug, sold by Clay & Co." Three tables for one screen.

Most of the time that's cheap and completely correct — databases are built to join. But joins have a cost, and a deeply normalized schema can need *many* of them to answer a single question. That cost is the temptation: someone looks at a slow page, sees five joins, and reaches for the scissors. Sometimes that's right. Usually it's premature. The rest of the chapter is about telling the difference.

## Section 3 — Denormalization: copying on purpose

**Denormalization** is deliberately storing a fact in more than one place — trading the cleanliness of "one source of truth" for something you want more. To see the danger first, here's denormalization done *carelessly*, "for speed": copy the vendor's name onto every product so the catalogue never has to join.

```
products  (denormalized carelessly — vendor_name copied onto every row)
──────────────────────────────────────────────────────
 id │ title        │ vendor_name   │ price_cents
────┼──────────────┼───────────────┼────────────
 …  │ Celadon mug  │ Clay & Co     │ 2000
 …  │ Stoneware bowl│ Clay & Co    │ 3500
 …  │ Mug set      │ Clay & Co     │ 5500     ← the same fact, copied 500 times
```

It reads fast — no join. Then Clay & Co renames their shop to "Clay & Company," and the anomaly strikes: you must now update *every one of their product rows in lockstep*. Update 499 and miss one, and your catalogue shows two different names for the same vendor — a customer-visible inconsistency, and a bug that's miserable to track down. **A stale copy is a bug.** That risk is the price of denormalization, and it's why normalization is the default.

## Section 4 — The two reasons to denormalize (the key idea)

Here's the insight that resolves the whole module. There are **two completely different reasons** to copy data, and they have *opposite* relationships with that staleness risk:

**Reason A — to preserve history (correctness over time).** This is your order snapshot. You copied the price and title onto the order line — but notice: that copy is *meant* to diverge from the live product. The price changes next week; the order must *not*. There is no "sync bug" here because **you never sync it** — divergence is the entire purpose. The order is a photograph of a moment, and photographs aren't supposed to update. Denormalizing for history is **always correct**, and carries none of Section 3's risk, because the thing you feared there (copies disagreeing) is here the thing you *want*.

**Reason B — to improve performance.** This is the vendor-name copy: you store a redundant fact purely so a read is faster (skip a join, pre-compute a count). Here, divergence *is* a bug — the copy is supposed to stay identical to the source — so you take on a real, ongoing duty to keep them in sync, or to accept staleness deliberately and with a plan. This is the risky kind, and you only do it **with evidence that you need it.**

> The same mechanism — copying a fact — is *correctness* in case A and a *liability* in case B. The difference is entirely intent: in A you copy because the value must be **frozen**; in B you copy because the join is **measured to be too slow**. Confusing the two is how people either fear the harmless snapshot or casually ship the dangerous copy.

## Section 5 — The decision framework

So the rule you carry forward:

1. **Normalize by default.** One fact, one place. This is where every table starts.
2. **Denormalize for history freely** — the snapshot, the frozen total. It's not an optimization, it's correctness; reach for it whenever a record must remember what *was* true.
3. **Denormalize for performance only with evidence** — a query you *measured* to be too slow, not one you *suspect* might be. And when you do, you own the sync.
4. **Never denormalize speculatively.** "This might be faster" is not a reason. A profiler showing a real, slow, frequent query is.

And one more, looking ahead: denormalizing for speed should usually be your *last* resort, not your first. Often an **index** (Chapter 38) makes the join fast enough that you never needed the copy at all — you fix the slow read without taking on the staleness risk. And **caching** (Chapters 40–41) is really denormalization-for-performance with an expiry stamped on it — a deliberately temporary copy with a plan for going stale. So the honest order of operations is: normalize → measure → index → *then*, if still needed, cache or denormalize. Copying data to go faster is a decision you earn, not one you assume.

## Section 6 — Prove it on your own project (hands-on)

Don't take any of this on faith — your seeded marketplace from Chapter 13 can demonstrate every claim. Run these against your own database and watch the principle hold.

**1. See the join cost of normalization.** A product page needs the product, its store, and the vendor — three normalized tables. Run the join and watch them reassembled:

```bash
psql "$DATABASE_URL" -c "
  SELECT p.title, s.name AS store, a.email AS vendor
  FROM products p
  JOIN stores   s ON s.id = p.store_id
  JOIN accounts a ON a.id = s.vendor_id
  LIMIT 5;"
```

That's normalization's price — three tables to show one row. (Tuck this query away; in Chapter 36 you'll learn to spot when a pattern like this fires *once per product* and becomes an N+1 problem.)

**2. Watch normalization make the update anomaly impossible.** Rename one of your seeded vendors' stores with a *single* update, then re-run the join — every product for that store now shows the new name, because the name lives in exactly one place:

```bash
psql "$DATABASE_URL" -c "UPDATE stores SET name = 'Clay & Company' WHERE name = 'Clay & Co';"
# re-run the query from step 1 → every Clay product reflects it instantly. One fact, one update.
```

Now imagine you'd copied that name onto every product (Section 3): the same rename would need hundreds of updates in lockstep, and any you missed would be wrong. You just felt *why* normalization is the default.

**3. Confirm your history-denormalization is correct.** Change a product's price and check an order that bought it — the order is unmoved, exactly as in Chapter 11:

```bash
psql "$DATABASE_URL" -c "UPDATE products SET price_cents = 9999 WHERE id = '<a sold product>';"
psql "$DATABASE_URL" -c "SELECT unit_price_cents FROM order_items WHERE product_id = '<that product>';"
# → still the original price. Divergence here is the FEATURE, not a bug.
```

**4. Write the artifact.** Add a short **`docs/SCHEMA-DECISIONS.md`** to your repo that classifies each major modelling choice — for each, one line: *normalized*, *denormalized-for-history*, or *denormalized-for-performance*, and why. You'll have:

- `accounts` / `stores` / `products` as separate FK-linked tables → **normalized** (one fact, one place).
- `order_items.unit_price_cents`, `title_snapshot`, `orders.total_cents` → **denormalized for history** (correct by design; meant to diverge).
- **Anything denormalized for *performance*?** → **none yet** — and that's exactly right, because you've taken no measurements. When Week 3 hands you a *measured* slow query, you'll revisit this doc with evidence.

Commit that file. It's a living record of your reasoning that the rest of the course (and any reviewer) can read — and the first entry in a habit of *documenting why*, not just *what*.

> **📖 Mandatory read — before Chapter 15.** Read a clear explainer on **database normalization (1NF, 2NF, 3NF in plain English)** and one on **when to denormalize** (search *"database normalization explained simply"* and *"when to denormalize database"*). *Required: the modules ahead — performance, caching — are largely about making these trade-offs deliberately, and they assume this vocabulary.*

> **Interesting to read.** The biggest read-heavy systems on the internet denormalize aggressively — social feeds, product catalogues at Amazon-scale — precisely because, *at that volume and with measurement*, the join cost is real and the sync cost is worth paying. Search *"denormalization at scale"* to see the disciplined version of what Section 3 warned against: same technique, but earned with data, and engineered to keep the copies honest.

## Key takeaways

You can now explain:

- **Normalization** means every fact lives in one place, and it's the default because it makes the **update anomaly** impossible.
- Its cost is **joins**, and that cost is the (often premature) temptation to denormalize.
- **Denormalization** copies a fact deliberately — risky when done for speed (a stale copy is a bug), and the reason normalization stays the default.
- There are **two reasons** to denormalize: for **history** (the snapshot — divergence is the point, always safe) and for **performance** (divergence is a bug — only with measured evidence and an owned sync).
- The order of operations for a slow read is **normalize → measure → index → then cache/denormalize**, never speculation first.

> **✍️ Log it (mandatory) — this is the gate.** In `learning-log/14-normalization-vs-denormalization.md` — **decision** first, then **topics**: **(Decision)** In your *own* words, why was it not a contradiction to normalize the whole schema and yet copy the price onto the order line — what makes those two choices both correct? Going forward, what would have to be true before you'd denormalize a part of *this* marketplace for speed? **(Topics)** (1) What is an *update anomaly*, and give a concrete example using the vendor-name-on-products case? (2) Explain the difference between denormalizing for *history* and for *performance*, and why only one carries a sync risk? (3) Why should indexing usually be tried *before* denormalizing for speed? (4) From the queries you ran and your `SCHEMA-DECISIONS.md`: what did the single-`UPDATE` rename demonstrate about normalization, and which of your decisions are denormalized-for-history?

*Answered all of it in your log? Then the data module is complete — you can model a marketplace and defend every modelling choice you made. Next we change gears entirely: making sure the data that arrives from the outside world is safe before it ever reaches these tables.*

---

Next: your tables trust their contents — but nothing has yet checked the data coming *in* from users. Time to guard the front door. → **[Chapter 15 — Validation: guarding the boundary](15-validation-concept.md)**
