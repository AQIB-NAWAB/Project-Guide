# Chapter 32 — Product create and edit

The store is open (Chapter 31), and across the last seventeen chapters you've built every guard a product endpoint needs: validation (16), the error contract (17), authentication (24), roles (26), scoping (28), and ownership (29). You've even built pieces of product create and edit along the way as vehicles for those lessons. This chapter pulls it all together into the **complete, coherent product-management feature** a vendor actually uses — create a listing, edit any field, and move it between draft and published — with every guard composed in the right order.

It's a consolidation chapter, and a good moment to make a clean API decision (how partial updates work) and to *feel* the payoff of earlier modelling choices — especially that editing a product's price leaves past orders untouched (Chapters 10, 14).

## Where we're headed

By the end a vendor has full lifecycle control of their own products — create as draft, edit any field including the JSONB attributes and price, publish and unpublish — all validated, owned, and scoped, with `PATCH` semantics decided deliberately.

## Step 1 — The product lifecycle

A product isn't created and frozen — it moves through states the vendor controls:

- **Draft** — created but not public; the vendor refines it. (Default `status` from Chapter 9.)
- **Published** — live in the catalogue, visible to customers.
- Back to **draft** (unpublish) or edited at any time.

This lifecycle is why `status` is a real column (Chapter 9) and why publishing is gated on a verified vendor (Chapter 23). The vendor drives it; your endpoints enforce who may, and on what.

## Step 2 — Create, completed

You built `POST /products` incrementally; here's the finished contract, with every guard in order:

```
POST /products   (requireAuth → requireRole('vendor') → requireVerified → validate(body))
  body: { title, price_cents, currency, status?, attributes? }   // NO store_id — derived from req.user
  → 201  the created product, in the caller's store, defaulting to draft
```

The store comes from `req.user` (Chapter 24), money is integer cents with a currency (Chapter 9), and `attributes` carries the type-specific JSONB tail (a mug's glaze, a scarf's fibre). Validation rejects a bad body with `400`; the vendor can only create in *their* store.

## Step 3 — Edit: PATCH vs PUT (the decision)

Editing raises a real API-design choice: when a vendor updates a product, do they send the *whole* product or just the changed fields?

| | `PUT` (full replace) | `PATCH` (partial update) |
|---|---|---|
| Client sends | The entire resource | Only the fields to change |
| Omitted field means | "set it to absent/default" | "leave it as-is" |
| Risk | Forget a field → wipe it | — |
| Fit for "change the price" | Awkward (resend everything) | Natural |

**The course uses `PATCH`** for product edits — a vendor changing just the price shouldn't have to resend the title, description, and every attribute (and risk wiping one they forgot). Validate a `PATCH` body as **all-fields-optional but each valid if present** (Chapter 16): you're validating the *changes*, not requiring the whole object. Ownership is enforced via the scoped mutation from Chapter 29 — the update only touches the row if it's the caller's, else `404`.

```
PATCH /products/:id   (auth → vendor role → validate(:id, partial body), scoped to req.user's store)
  body: { price_cents?: ..., status?: 'published', attributes?: ... }   // only what changes
  → 200 updated  |  404 if not the caller's product
```

## Step 4 — Publish is just an edit — but feel what it does *not* touch

Publishing is `PATCH`ing `status` to `published` (and unpublishing the reverse). No special endpoint needed — it's a field change through the same guarded path. But pause on a subtle, important consequence you designed for chapters ago:

When a vendor **edits the price** of a product that has already sold, the change applies to the *live* product only. Every existing order keeps the price it recorded at purchase, because order lines **snapshotted** it (Chapter 10) — denormalization-for-history (Chapter 14). Ownership controls who can change the catalogue; snapshots keep the books honest. This is the moment those earlier decisions visibly pay off, and it's worth having the vendor *see* it.

## Step 5 — Do it on your project (hands-on)

**1. Finalise `POST /products`** with the full guard chain (auth → role → verified → validate), store derived from `req.user`, defaulting to draft.

**2. Build `PATCH /products/:id`** with partial validation and scoped ownership (Chapter 29): updates only the caller's product, `404` otherwise. Support editing title, description, price, currency, `attributes`, and `status`.

**3. Walk the full lifecycle and prove the snapshot:**

```bash
V=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"maker@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)

# create (draft) → edit attributes → publish
PID=$(curl -s -X POST localhost:3000/products -H "authorization: Bearer $V" -d '{"title":"Celadon mug","price_cents":2000,"currency":"USD","attributes":{"glaze":"celadon"}}' -H 'content-type: application/json' | jq -r .id)
curl -i -X PATCH localhost:3000/products/$PID -H "authorization: Bearer $V" -d '{"attributes":{"glaze":"celadon","capacity_ml":350}}' -H 'content-type: application/json'   # → 200
curl -i -X PATCH localhost:3000/products/$PID -H "authorization: Bearer $V" -d '{"status":"published"}' -H 'content-type: application/json'                                    # → 200 published

# raise the price, then check a PAST order that bought this product
curl -i -X PATCH localhost:3000/products/$PID -H "authorization: Bearer $V" -d '{"price_cents":2500}' -H 'content-type: application/json'                                      # → 200
psql "$DATABASE_URL" -c "SELECT unit_price_cents FROM order_items WHERE product_id = '$PID';"   # → still 2000. History intact.
```

The vendor controls the live product; the order remembers what was paid. That's the data model and the security model working as one.

> 💡 **Hint — soft-delete, don't hard-delete, a sold product.** Recall Chapter 11: a product that's been sold can't be hard-deleted without orphaning order history (the `RESTRICT`). So "delete" for a vendor should usually be a **soft delete** (`status = 'archived'` or a `deleted_at`) that hides it from the catalogue but preserves it for past orders. Wire your `DELETE` to soft-delete sold products (and only hard-delete never-sold drafts, if you allow it at all).

> **📖 Mandatory read — before Chapter 33.** Read a short piece on **PATCH vs PUT semantics** (search *"PATCH vs PUT REST"*). *Required: partial-update semantics shape every edit endpoint you'll write.*

> **Interesting to read.** "Editing the price shouldn't rewrite history" feels obvious once stated, yet real systems have shipped bugs where changing a catalogue price retroactively altered customers' past invoices. Search *"price change affected past orders bug"* — your snapshot design (Chapter 10) is the standard fix, and you just watched it hold.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 33:

- [ ] `POST /products` creates in the caller's store with the **full guard chain**, defaulting to draft, no `store_id` from the client
- [ ] `PATCH /products/:id` does **partial updates** (all fields optional, each validated), scoped to the owner (`404` otherwise)
- [ ] A vendor can edit any field — including JSONB `attributes` — and **publish/unpublish** via `status`
- [ ] Editing a product's price leaves **existing orders' snapshots unchanged** (you verified)
- [ ] "Delete" **soft-deletes** a sold product rather than orphaning order history
- [ ] You can justify choosing `PATCH` over `PUT` for edits

> **✍️ Log it (mandatory).** In `learning-log/32-product-create-edit.md` — **decision** first, then **topics**: **(Decision)** Why `PATCH` over `PUT` for product edits, and what would the wrong choice risk? Why must "delete" be a soft delete for a sold product? **(Topics)** (1) List the guard chain on create/edit, in order, and what each enforces. (2) How does partial-update validation differ from create validation? (3) Why does editing the live price not change past orders — which earlier decisions make that true? (4) Why is publishing "just an edit" rather than a special endpoint?

*All boxes ticked and the log written? Then continue. A vendor can manage products one at a time — now give them the list view that shows their whole shop at a glance.*

---

Next: build the vendor's full product list — their own catalogue, filtered and sorted, the data behind their dashboard. → **[Chapter 33 — The vendor product list](33-vendor-product-list.md)**
