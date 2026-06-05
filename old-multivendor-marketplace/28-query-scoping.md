# Chapter 28 — Query scoping

Your `ISOLATION.md` (Chapter 27) names every vendor-owned resource and its scope key. Now you build the first and most important layer of isolation: **scope every query to the current tenant**, so vendor A's queries can only ever return vendor A's rows. Done right, another tenant's data isn't *forbidden* — it's *invisible*, never coming back from the query at all.

The danger this guards against is subtle and one-sided: you have to get scoping right on *every* query, and forgetting it on *one* is a leak. So the real work of this chapter isn't a clever query — it's making the scope the **default path**, something the structure of your code applies for you, rather than a line you must remember each time.

## Where we're headed

By the end, reads and writes of vendor-owned data carry a tenant filter by default; you have a small scoping helper so the filter can't be forgotten; a vendor's product list returns only their own products; and you've proven two vendors can't see each other's data.

## Step 1 — The leak hiding in an ordinary query

Here's a perfectly normal-looking handler for "get a product to edit":

```sql
-- ❌ global query — returns ANY product, whoever owns it
SELECT * FROM products WHERE id = $1;
```

Nothing about it is *wrong* as SQL. The problem is what it *omits*: it asks "the product with this id," not "*my* product with this id." Vendor A passes vendor B's product id and gets B's product back. The same omission on an `UPDATE` or `DELETE` lets A *modify* B's data. This single missing clause is the entire IDOR vulnerability.

The fix is to make every such query ask the scoped question:

```sql
-- ✅ scoped query — returns the product ONLY if it belongs to my store
SELECT * FROM products WHERE id = $1 AND store_id = $2;   -- $2 = the caller's store
```

Now if the product isn't yours, the query returns *zero rows* — and your handler treats that exactly like "not found." A non-owned object is indistinguishable from a non-existent one, which (bonus) also avoids leaking whether it exists at all (the enumeration lesson from Chapter 20).

## Step 2 — Where the scope comes from

The scope is never from the request — it's derived from the **verified identity**. `requireAuth` (Chapter 24) gave you `req.user.id` (the vendor); from that you resolve their store (one store per vendor, Chapter 8), and *that* store id is your scope key. The client never sends it and can never override it. This is the same principle as deriving the store on create (Chapter 24): the tenant key comes from the token, full stop.

## Step 3 — Make scoping the default, not a thing you remember

Relying on yourself to hand-write `AND store_id = $2` on every query is exactly how a leak eventually happens — one busy afternoon, one forgotten clause. So build the scope *into the path*. The shape depends on your stack, but the idea is constant: **a scoped data-access helper that always applies the tenant filter**, so vendor-owned data can only be reached *through* it.

```
// the CONTRACT of a scoped accessor — you implement it over your DB layer
forVendor(storeId).products.findById(id)     // always adds AND store_id = storeId
forVendor(storeId).products.list(filter)     // always scoped
forVendor(storeId).products.update(id, data) // update ... WHERE id = id AND store_id = storeId
```

The win is that the *unscoped* query becomes the one you have to go out of your way to write, instead of the default. Code review and your own habits then have a single, visible rule: vendor data goes through `forVendor(...)`. (If you later add Postgres Row-Level Security as the Chapter 27 backstop, it enforces this even when a query slips — but the app-level default is your first line.)

## Step 4 — Apply it: the vendor's own product list

Use the scope to build the vendor's view of their products — `GET /vendor/products`, returning only the caller's store's products:

```
GET /vendor/products            (requireAuth + requireRole('vendor'))
  → server resolves req.user → their store → list products WHERE store_id = <their store>
  → returns only THIS vendor's products (drafts and published), never anyone else's
```

Notice this is the *opposite* of the public catalogue you'll build in Chapter 35 (which spans all stores): the vendor list is tightly scoped to one tenant. (A full version with filters and sorting is Chapter 33; here, a basic scoped list is enough to prove isolation.)

## Step 5 — Do it on your project (hands-on)

**1. Build a scoping helper** (Step 3) — a thin accessor that takes the caller's store id and applies `store_id = <that>` to every product read and write. Make it the *only* way handlers reach vendor-owned product data.

**2. Add `GET /vendor/products`**, scoped to `req.user`'s store via the helper.

**3. Scope the existing `GET /products/:id` for vendor access** so fetching a product to edit goes through the scoped accessor (returns nothing if it isn't yours). (The mutation endpoints get ownership enforcement in Chapter 29.)

**4. Prove isolation with two tenants:**

```bash
# log in as vendor A and vendor B (two seeded vendors)
A=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"a@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)
B=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"b@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)

curl -s -H "authorization: Bearer $A" localhost:3000/vendor/products | jq '.[].title'   # → only A's products
curl -s -H "authorization: Bearer $B" localhost:3000/vendor/products | jq '.[].title'   # → only B's products (no overlap)

# A tries to fetch one of B's products by id → behaves as "not found"
curl -i -H "authorization: Bearer $A" localhost:3000/products/<B's-product-id>           # → 404
```

Two vendors, two disjoint lists, and A reaching for B's product gets *nothing* — that's Layer 1 working. The disjointness is the property Chapter 30 will lock in with an automated test.

> 💡 **Hint — return 404, not the row, for "not yours."** When a scoped query comes back empty because the object belongs to someone else, respond `404` (or your "not found" path) — never a `403` that admits "this exists but isn't yours," and never the row. Empty-means-absent keeps cross-tenant existence hidden and keeps your handlers simple: scoped query → no row → not found.

> **📖 Mandatory read — before Chapter 29.** Read your ORM/query-builder's docs on **reusable query scopes / default filters** (most have a pattern for this), and revisit the **OWASP BOLA** guidance. *Required: Chapter 29 builds the explicit ownership check for mutations on top of this scoping.*

> **Interesting to read.** Many multi-tenant data leaks come down to exactly one un-scoped query in a large codebase — which is why mature SaaS teams enforce scoping structurally (a default tenant filter, or database row-level security) rather than trusting every developer to remember it on every query. Search *"multi-tenant missing tenant filter data leak"*.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 29:

- [ ] Vendor-owned product reads/writes go through a **scoping helper** that always applies `store_id = <caller's store>`
- [ ] The scope is derived from **`req.user`** (the verified token), never from the request
- [ ] `GET /vendor/products` returns **only the caller's** products; two vendors see **disjoint** lists (you verified)
- [ ] Vendor A fetching vendor B's product by id behaves as **not found** (`404`), not a `403` and not the row
- [ ] You can explain why the unscoped query is the *harder* one to write in your code now

> **✍️ Log it (mandatory).** In `learning-log/28-query-scoping.md` — **decision** first, then **topics**: **(Decision)** Why make scoping the *default path* via a helper rather than adding `AND store_id = ...` by hand each time? Why does the scope come from `req.user` and never the request? **(Topics)** (1) Show the difference between the unscoped and scoped product query and what each returns for a non-owned id. (2) Why respond `404` rather than `403` (or the row) for "exists but not yours"? (3) How does a scoping helper make a forgotten scope *less* likely? (4) How does query scoping differ from the role checks you built in Chapter 26?

*All boxes ticked and the log written? Then continue. Reads are scoped — but operations that target a specific object by id (edit, delete) need an explicit ownership check. Next you close the IDOR hole for good.*

---

Next: scoping hides other tenants' rows — now enforce ownership on the operations that act on a specific object, closing the IDOR hole from Chapter 26. → **[Chapter 29 — Ownership and IDOR](29-ownership-and-idor.md)**
