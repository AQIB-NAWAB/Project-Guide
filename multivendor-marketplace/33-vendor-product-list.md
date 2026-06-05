# Chapter 33 — The vendor product list

A vendor can now create, edit, and publish products one at a time (Chapter 32). But to *run* a shop, they need the bird's-eye view: all their listings in one place, filterable by status, sortable, so they can find the draft they were working on or see what's live. You built a minimal scoped list back in Chapter 28 to demonstrate isolation; this chapter turns it into the real, usable **vendor product list** — the data behind the dashboard you'll build next.

It's a focused build that introduces query features you'll use everywhere: filtering and sorting from validated query parameters. It also surfaces a problem you'll deliberately *not* fully solve yet — what happens when a vendor has thousands of products — setting up the performance module in Week 3.

## Where we're headed

By the end `GET /vendor/products` returns the caller's products, filterable by status (draft/published/archived) and sortable, all from validated query params — scoped so a vendor only ever sees their own.

## Step 1 — Scoped by default, always

The non-negotiable foundation, carried from Chapter 28: this list is **scoped to the caller's store**, through your scoping helper. A vendor's product list can *never* include another vendor's products — that's the isolation you built and tested (Chapter 30), and every query here goes through `forVendor(...)`. Everything else in this chapter — filters, sorts — layers *on top* of that scope, never replaces it.

## Step 2 — Filtering from validated query params

A vendor wants to see "just my drafts" or "just what's published." That's **filtering** via query parameters — `GET /vendor/products?status=draft` — and query strings are untrusted input (Chapter 15), so they go through validation (Chapter 16):

```
GET /vendor/products?status=draft&sort=created_at:desc
  validate(query): {
    status?: 'draft' | 'published' | 'archived'   // enum — reject anything else with 400
    sort?:   one of an allowlist of sortable fields + direction
  }
  → the caller's products matching the filter
```

Two things to get right. **Filters are an allowlist**, exactly like the body schema (Chapter 16): `status` must be one of the known values, or `400` — never pass a raw query value into a query. And the filter **composes with the scope**: `WHERE store_id = <mine> AND status = 'draft'` — the tenant filter is always there; the status filter is added.

## Step 3 — Sorting safely (an injection footgun)

Sorting feels trivial — `ORDER BY <field>` — but it's a classic injection hole if you let the client name the column directly. A request with `?sort=password_hash` or a crafted value could read or break things if you interpolate it into SQL. **The defence is an allowlist again:** map the client's `sort` value to a fixed set of permitted columns and directions you control; reject anything else. The client picks *from* your list; it never *supplies* a column name.

```
// ✅ allowlist mapping — client value → safe, known column
sortable = { 'created_at:desc': ..., 'price:asc': ..., 'title:asc': ... }
// a sort value not in this map → 400, never reaches the query
```

This is the same principle as validation everywhere: the client expresses intent within options *you* define; raw client strings never become SQL identifiers.

## Step 4 — The volume problem (named, not yet solved)

Return the vendor's *entire* product list and it's fine at ten products, seed-data-sized. But a successful maker has hundreds, and a query that returns *everything* — plus rendering it all — gets slow and wasteful as the shop grows. You can feel the shape of three problems coming: fetching too many rows at once (**pagination**, Chapter 37), the related-data lookups when you show each product's details (**N+1**, Chapter 36), and slow filtering/sorting without the right **indexes** (Chapter 38).

Don't solve them now — name them. This chapter delivers a correct, scoped, filterable list; Week 3 makes it *fast* at scale. (Recall the Chapter 14 discipline: build it correct and normalized first, then *measure* before optimizing. You're at the "correct" stage.)

## Step 5 — Do it on your project (hands-on)

**1. Build the full `GET /vendor/products`** through your scoping helper, accepting validated `status` and `sort` query params (allowlisted), composing them onto the tenant scope.

**2. Test filtering, sorting, and isolation together:**

```bash
A=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"a@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)

curl -s -H "authorization: Bearer $A" "localhost:3000/vendor/products?status=draft"          | jq '.[].title'   # → A's drafts only
curl -s -H "authorization: Bearer $A" "localhost:3000/vendor/products?sort=price:desc"        | jq '.[].price_cents'  # → A's products, priciest first
curl -i -H "authorization: Bearer $A" "localhost:3000/vendor/products?status=not_a_status"                       # → 400 (allowlist)
curl -i -H "authorization: Bearer $A" "localhost:3000/vendor/products?sort=password_hash"                        # → 400 (sort allowlist)
```

Filters and sorts work, bad values are rejected with `400`, and the list never leaves the vendor's tenant. Confirm a second vendor's list is still completely disjoint (the Chapter 30 test should stay green).

> 💡 **Hint — never interpolate a query param into SQL identifiers.** A filter *value* goes in as a parameterized value (`= $1`); a sort *column* or direction must be chosen from your allowlist and mapped to a literal you wrote. The moment a client-supplied string could become a column name or SQL keyword, you have an injection bug — the allowlist is what makes sorting safe.

> **📖 Mandatory read — before Chapter 34.** Read a short piece on **safe dynamic sorting/filtering (allowlisting columns)** and **SQL injection via ORDER BY** (search *"order by sql injection allowlist"*). *Required: filtering and sorting are everywhere from here on, and this is the trap to avoid.*

> **Interesting to read.** `ORDER BY` injection is a genuinely overlooked vector — parameterized queries protect *values* but not *identifiers*, so an allowlisted sort column is the only safe way. Search *"sql injection order by clause"* to see why "it's just sorting" has caused real vulnerabilities.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 34:

- [ ] `GET /vendor/products` returns the **caller's** products through the scoping helper — isolation intact
- [ ] **Filtering** by `status` works and is **allowlisted** (a bad value → `400`)
- [ ] **Sorting** is **allowlisted** — a client can't sort by an arbitrary/sensitive column (→ `400`)
- [ ] Filters and sorts **compose with the tenant scope**, never replace it
- [ ] You can name the three performance problems (pagination, N+1, indexing) this list will face at scale and that Week 3 addresses

> **✍️ Log it (mandatory).** In `learning-log/33-vendor-product-list.md` — **decision** first, then **topics**: **(Decision)** Why allowlist sortable columns instead of accepting the client's `sort` value directly — what's the risk? Why is this list scoped before any filter is applied? **(Topics)** (1) How does a filter value enter the query differently from a sort column, and why? (2) What is `ORDER BY` injection and how does an allowlist prevent it? (3) Why build this list "correct first" and defer performance to Week 3 (tie to Chapter 14's discipline)? (4) Which three scaling problems will this endpoint face, and which chapters address them?

*All boxes ticked and the log written? Then continue. The vendor's data is all there and secure — now put a face on it with a dashboard they actually use.*

---

Next: assemble everything into the vendor's dashboard — the frontend that drives login, products, and the API you've built. → **[Chapter 34 — The vendor dashboard (frontend)](34-frontend-vendor-dashboard.md)**
