# Chapter 45 — Order splitting

The cart holds a mug from Clay & Co and a scarf from WoolWorks (Chapter 44). The customer hits "checkout" and pays once. Now comes the decision that *defines* a marketplace, the one you modelled all the way back in Chapter 10: that single checkout is **not one order** — it's **one order per vendor**, because each maker fulfils and is paid separately. This chapter builds the splitting logic that turns a multi-vendor cart into the correct set of orders.

It's the first of four chapters that build checkout incrementally — here, the *split*; then snapshots (46), the transaction (47), and idempotency (48). Keeping them separate lets each correctness concern get the attention it deserves.

## Where we're headed

By the end, checkout groups the cart's items by vendor and creates **one order per vendor** under a shared `checkout_id`, each containing only that vendor's items — the structure you designed in Chapter 10, now real.

## Step 1 — Why split (the decision, revisited)

You reasoned this through in Chapter 10, so here's the short version with the *why* fresh: one customer's payment, several makers. Each vendor ships their own items, is paid for their own items, and tracks their own fulfilment status. A single shared order can't represent two makers moving independently, and it would breach isolation (a vendor seeing another's items in "their" order). So:

```
Checkout (one payment)
 ├─ Order → Clay & Co     (the mug)      status, total, payout — independent
 └─ Order → WoolWorks     (the scarf)    status, total, payout — independent
   linked by a shared checkout_id
```

The `checkout_id` (Chapter 10's `orders.checkout_id`) ties the sibling orders together as "one purchase" for the customer's view, while each order stands alone for the vendor.

## Step 2 — The split, step by step

Given the customer's cart, the algorithm is:

1. **Load and validate the cart** — its items, and re-check each product is still available and published (Chapter 44).
2. **Group items by vendor** — partition cart items by their product's `store_id`. Each group becomes one order.
3. **Create one order per group** — an `orders` row per vendor (`store_id`, `customer_id`, a shared `checkout_id`, initial status), with its `order_items` (Chapter 46 will snapshot the prices onto them).
4. **Clear the cart** once orders exist.

```
cart items → group by store_id → for each group: create an order + its order_items → clear cart
```

This chapter focuses on producing the *right structure*. The next three make it *correct under failure*: snapshotting what was paid (46), making the whole thing all-or-nothing (47), and safe against double-submit (48). Build the split now; you'll harden it immediately after.

## Step 3 — Each order is owned and isolated

The orders you create participate in the same isolation model as everything else (Chapters 27–30): each order's `store_id` ties it to one vendor (so that vendor — and only that vendor — sees it in their order list), and `customer_id` ties it to the buyer (so the customer sees all their orders across vendors). You're not inventing new access rules — checkout produces data that *fits* the isolation you already built and tested.

## Step 4 — Do it on your project (hands-on)

**1. Build `POST /checkout`** (auth + customer): load the caller's cart, validate availability, group items by `store_id`, and create one order (with its order items) per vendor under a single generated `checkout_id`, then clear the cart.

**2. Test the split with a multi-vendor cart:**

```bash
C=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"buyer@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)
# (cart already has a Clay & Co mug and a WoolWorks scarf from Chapter 44)

curl -s -X POST localhost:3000/checkout -H "authorization: Bearer $C" | jq '{checkoutId, orders: [.orders[].storeId]}'
# → ONE checkoutId, TWO orders — one per vendor

# verify: each order contains only its vendor's items, and vendors see only their own order
psql "$DATABASE_URL" -c "SELECT store_id, count(*) FROM orders WHERE checkout_id='<id>' GROUP BY store_id;"   # → 2 orders, 2 stores
```

Two vendors in the cart producing two orders under one checkout — each containing only that vendor's items — is the split working. Confirm via the vendor order view (each vendor sees only their order) and the customer view (sees both).

> 💡 **Hint — a single-vendor cart still splits into "one" order, and that's fine.** Don't special-case the count. The algorithm is "group by vendor, one order per group" — a cart with items from one vendor produces one order; from three vendors, three orders. Writing it as a general grouping (rather than "if multiple vendors then split") keeps it correct for every case and matches how you'll reason about payouts later.

> **📖 Mandatory read — before Chapter 46.** Revisit your own **Chapter 10** notes and read about **marketplace order models / split orders** (search *"marketplace split order per seller"*). *Required: Chapter 46 snapshots prices onto the order items this split creates.*

> **Interesting to read.** Every major marketplace (Etsy, Amazon's third-party sellers) splits a multi-seller cart into per-seller orders behind the scenes — one payment, many fulfilments and payouts. Search *"how marketplaces split orders by seller"* to see the pattern you just built is the universal one.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 46:

- [ ] `POST /checkout` groups the cart by vendor and creates **one order per vendor**, each with only that vendor's items
- [ ] Sibling orders share a **`checkout_id`**; each order has its own `store_id` and the buyer's `customer_id`
- [ ] A **multi-vendor cart produces multiple orders**; a single-vendor cart produces one — same general logic
- [ ] Created orders fit the **isolation model** (vendor sees own; customer sees all theirs)
- [ ] The cart is **cleared** after orders are created
- [ ] (Hardening — snapshots, atomicity, idempotency — comes in Chapters 46–48)

> **✍️ Log it (mandatory).** In `learning-log/45-order-splitting.md` — **decision** first, then **topics**: **(Decision)** Why does one checkout become one order *per vendor* rather than a single order — name the things that break otherwise? Why write the split as general grouping rather than special-casing multi-vendor? **(Topics)** (1) Walk the split algorithm step by step. (2) What does `checkout_id` tie together, and what does each order own independently? (3) How do the new orders fit the isolation model? (4) Why must cart items be re-validated at checkout?

*All boxes ticked and the log written? Then continue. The orders exist — but their line items don't yet record *what was actually paid*. Time to freeze the prices.*

---

Next: the cart showed live prices — checkout must record what was actually paid, immune to later changes. → **[Chapter 46 — Price snapshots](46-price-snapshots.md)**
