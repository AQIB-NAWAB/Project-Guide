# Chapter 46 — Price snapshots

Checkout now creates the right *structure* — one order per vendor (Chapter 45). But there's a hole in it: the order items don't yet record *what the customer actually paid*. The cart showed **live** prices (Chapter 44), deliberately. The order must record **frozen** prices — because the instant the customer pays, the price for *that sale* is fixed forever, no matter what the vendor does to the product tomorrow. This is the **price snapshot** — the idea you modelled in Chapter 10, enforced with constraints in Chapter 11, and now finally *create* at the one moment it's meant to happen: purchase.

It's a short chapter with an outsized correctness payoff. Get it right and your orders are honest historical records; get it wrong and editing a price silently rewrites customers' receipts.

## Where we're headed

By the end, checkout copies each product's current price and title onto its `order_item` at the moment of sale, computes frozen line and order totals, and you've proven a later price change doesn't touch the completed order.

## Step 1 — The moment to freeze is *now*

Recall the cart-vs-order line from Chapter 44: the cart is live (shows current price), the order is frozen. **Checkout is the boundary** where live becomes frozen. As you create each `order_item` during the split (Chapter 45), you copy from the product *as it is right now*:

- `unit_price_cents` ← the product's **current** `price_cents`
- `title_snapshot` ← the product's **current** title
- `line_total_cents` ← `unit_price_cents × quantity`

And the order's `total_cents` ← the sum of its line totals. After this moment, the order *never reads the live product for price again* — it reads its own snapshot. The product can be edited, repriced, renamed, even archived; the order remembers exactly what was bought and paid (the `RESTRICT`/snapshot protections from Chapter 11 guarantee it stays referenceable).

## Step 2 — Why this is the correct design (one more time, concretely)

You've met this reasoning, but it lands hardest here, where the money is real. If an order item stored only `product_id` and read the price live at display time, then a vendor raising the mug from $20 to $25 next week would make *every past order* that bought it suddenly show $25 — wrong totals, wrong receipts, broken accounting (Chapter 10's exact failure). The snapshot is what makes the order a *record of a sale* rather than a *guess about the present*. It's denormalization-for-history (Chapter 14): the copy is *meant* to diverge from the live product, so there's no sync bug — divergence is the point.

## Step 3 — Handle the cart-vs-checkout price gap

One real edge case the snapshot surfaces: between viewing the cart (live price) and completing checkout, a vendor could change the price. Which price does the customer pay? Decide and be consistent — the common, fair choice is: **the price at the moment of checkout is authoritative**, and that's exactly what you snapshot. Optionally, if the price changed since the cart was last shown, surface it ("the price of this item changed") before charging. Either way, the snapshot at checkout is the single source of truth for what was paid — there's no ambiguity once it's frozen.

## Step 4 — Do it on your project (hands-on)

**1. Snapshot during the split.** In `POST /checkout` (Chapter 45), when creating each `order_item`, copy the product's current `price_cents` → `unit_price_cents` and title → `title_snapshot`, compute `line_total_cents`, and set each order's `total_cents` to the sum of its lines.

**2. Prove the freeze holds:**

```bash
# checkout a product, then read the order line's recorded price
curl -s -X POST localhost:3000/checkout -H "authorization: Bearer $C" | jq '.orders'
psql "$DATABASE_URL" -c "SELECT title_snapshot, unit_price_cents, line_total_cents FROM order_items WHERE order_id='<id>';"   # → the price AT CHECKOUT

# now the vendor RAISES the price
curl -s -X PATCH localhost:3000/products/$PID -H "authorization: Bearer $V" -d '{"price_cents":9999}' -H 'content-type: application/json'

# the completed order is UNCHANGED
psql "$DATABASE_URL" -c "SELECT unit_price_cents, total_cents FROM order_items oi JOIN orders o ON o.id=oi.order_id WHERE oi.product_id='$PID' AND o.id='<id>';"
# → still the original price. The sale is history; history doesn't move.
```

Watching the order keep its original price after the product's price jumps is the snapshot doing its one job. This is the third time you've proven it (Chapters 11, 32, now at the real checkout) — because it's the single most important correctness property in commerce.

> 💡 **Hint — snapshot everything the receipt needs, not just the price.** A receipt or order page must render correctly *forever*, even if the product is later deleted (Chapter 11's soft-delete world). So snapshot enough to display the line without the live product: at minimum the title and unit price; consider the product's key attributes too if your order view shows them. The test from Chapter 10 still applies — "delete the product in your head; does the order still render?"

> **📖 Mandatory read — before Chapter 47.** Re-read your **Chapter 10** snapshot reasoning and a piece on **immutable financial records** (search *"why orders snapshot price immutable"*). *Required: Chapter 47 wraps this snapshotting plus the split into one atomic transaction.*

> **Interesting to read.** Accounting and audit standards effectively *require* this — a sale is a fact at a point in time, and systems that let past transactions mutate fail audits. Search *"immutable transaction records accounting"* to see why "freeze what was paid" isn't just convenient, it's often a compliance requirement.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 47:

- [ ] Checkout **snapshots** each product's current price and title onto its `order_item` (`unit_price_cents`, `title_snapshot`)
- [ ] Line totals and the order `total_cents` are **computed and frozen** at checkout from the snapshots
- [ ] A vendor changing a product's price **after** checkout leaves the completed order **unchanged** (you verified)
- [ ] The order can render **without** reading the live product (enough is snapshotted)
- [ ] You can explain why this is denormalization-for-history, not a sync bug

> **✍️ Log it (mandatory).** In `learning-log/46-price-snapshots.md` — **decision** first, then **topics**: **(Decision)** Why is checkout the moment to freeze prices, and why is the *checkout-time* price authoritative over the cart-time price? **(Topics)** (1) What exactly gets snapshotted onto an order item, and why each field? (2) Walk the failure that storing only `product_id` would cause. (3) Why is this denormalization-for-history and therefore free of sync risk? (4) What's the "delete the product in your head" test, and does your order pass it?

*All boxes ticked and the log written? Then continue. Checkout creates orders and snapshots prices — but it's *many* writes that must all succeed together. Next you make it all-or-nothing.*

---

Next: checkout does several writes — orders, items, cart clearing — that must never half-happen. Wrap them in a transaction. → **[Chapter 47 — The checkout transaction](47-checkout-transaction.md)**
