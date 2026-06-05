# Chapter 35 — The public catalogue

Week 2 built the vendor's secured, isolated world. Week 3 turns to the *other* side of a marketplace — the **customer**, browsing. The public catalogue is the storefront: every *published* product from *every* maker, in one browsable list, open to anyone (no login required). It's the opposite of the vendor product list (Chapter 33): where that was tightly scoped to one tenant, the catalogue deliberately spans *all* stores.

It's a straightforward feature to build — and it's where the performance story of Week 3 begins, because a query across every product in the marketplace is the first one that gets *slow* at real scale. This chapter builds the catalogue correctly; the next four make it fast. (That order is the Chapter 14 discipline: correct first, then measure, then optimize.)

## Where we're headed

By the end you have a public `GET /products` catalogue that lists *published* products across all stores with their store name, excludes drafts and archived items, requires no auth, and leaks no private data — built correctly, ready to be measured and optimized.

## Step 1 — The catalogue is cross-tenant *by design*

This is the one place the tenant boundary intentionally opens. The vendor list (Chapter 33) answers "*my* products"; the catalogue answers "*all published* products, whoever made them." So the scoping flips: instead of `WHERE store_id = <mine>`, the catalogue filters by **visibility**, not ownership:

```
GET /products   (public — no auth)
  → published products from ALL stores, each with its store's display name
```

But "cross-tenant" must never mean "leaks private data." Two hard rules:

- **Only `status = 'published'`.** Drafts and archived products are private to their vendor — they must *never* appear in the catalogue. This is a visibility filter every catalogue query carries, the way vendor queries always carried the tenant scope.
- **Only public fields.** The catalogue shows what a shopper should see — title, price, store name, attributes, image — and *nothing* internal (no vendor email, no cost notes, no draft-only fields). Shape the response deliberately; don't just dump the row.

## Step 2 — Joining product to store for display

A catalogue entry shows "Celadon Mug — *Clay & Co*," so each product needs its **store's** display name. That's a join across the ownership chain you modelled in Chapter 8: `products → stores`. One row of the catalogue is a product plus a little of its store:

```
catalogue item:  { title, price_cents, currency, attributes, store: { name, slug } }
   from:         products JOIN stores ON stores.id = products.store_id  WHERE products.status = 'published'
```

This join is innocent-looking and correct — but *how* you fetch the store for each product is exactly where the next chapter's problem hides. Build it as a proper join now; Chapter 36 shows what goes wrong if you instead fetch each product's store one-by-one.

## Step 3 — Public means a different set of concerns

Because the catalogue is open to the world, a few things shift:

- **No auth, but still validated.** Query params (filters, later pagination) are still untrusted input — validate them (Chapter 16). Open ≠ unvalidated.
- **Don't show unverified or closed shops' products** if your rules require it (a sensible default: only products from verified vendors with open stores appear).
- **It will be your highest-traffic endpoint.** Everyone browses; few log in. That's *why* it's the one you'll cache (Chapters 40–41) and index (Chapter 38) most carefully — popularity makes its performance matter most.

## Step 4 — Do it on your project (hands-on)

**1. Build `GET /products`** (public, no auth) returning **published** products across all stores, each joined to its store's display name, with a deliberately shaped public response (no private fields).

**2. Verify visibility and cross-tenant correctness:**

```bash
# public — no token needed
curl -s localhost:3000/products | jq '.[] | {title, store: .store.name}'   # → published products from MANY stores

# a draft must NOT appear
# (publish one product, leave another as draft, confirm only the published one is listed)
curl -s localhost:3000/products | jq '.[].title'    # → includes published titles, excludes drafts
```

Confirm: products from multiple vendors appear, drafts/archived never do, and no private field (vendor email, internal notes) is in the response.

**3. Seed enough volume to make Week 3 real.** Use your Chapter 13 seed (its optional bulk generator) to create *hundreds to thousands* of published products across many stores. You need real volume, because the performance problems of the next chapters are invisible on ten rows and obvious on ten thousand.

> 💡 **Hint — shape the response, don't leak the row.** It's tempting to `SELECT *` and return it. Don't — a row may carry fields a shopper shouldn't see, and returning the raw row also couples your API to your schema. Map each product to an explicit public shape. This is the read-side mirror of the mass-assignment lesson (Chapter 15): control what goes *out* as deliberately as what comes *in*.

> **📖 Mandatory read — before Chapter 36.** Read a short primer on **SQL joins** (if you want a refresher) and how your ORM/query layer loads **related records**. *Required: Chapter 36 is entirely about the right and wrong ways to load the store for each catalogue product.*

> **Interesting to read.** The product catalogue / search page is almost always the most-hit, most-optimized endpoint in any e-commerce system — entire caching and indexing strategies exist just for it. Search *"ecommerce catalog page performance"* to see why the page you just built gets the most engineering attention of any in the app.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 36:

- [ ] `GET /products` is **public** (no auth) and lists **published** products across **all** stores
- [ ] **Drafts and archived products never appear**; only published items are visible
- [ ] Each item includes its **store's display name** (joined from `stores`)
- [ ] The response exposes **only public fields** — no vendor email or internal data
- [ ] Query params are still **validated** despite the endpoint being open
- [ ] You've **seeded real volume** (hundreds+ of products) to make Week 3's performance work meaningful

> **✍️ Log it (mandatory).** In `learning-log/35-public-catalogue.md` — **decision** first, then **topics**: **(Decision)** Why does the catalogue deliberately span all tenants when everything in Week 2 was about scoping to one — and what *replaces* the tenant scope as its safety rule? Why shape the response instead of returning the row? **(Topics)** (1) What two visibility rules must every catalogue query enforce? (2) Why does each catalogue item need a join to `stores`? (3) Why is "public" not the same as "unvalidated"? (4) Why seed large volume *before* the performance chapters?

*All boxes ticked and the log written? Then continue. Your catalogue works — but the way it loads each product's store is about to become its biggest performance problem. Time to measure.*

---

Next: the catalogue works on seed data — now measure it at volume and meet the most famous performance bug in web development. → **[Chapter 36 — The N+1 query problem](36-n-plus-1.md)**
