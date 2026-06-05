# Chapter 44 — The cart

Customers can browse a fast, searchable, photo-rich catalogue (Chapters 35–43). Now let them *buy* — and that starts with a cart, the place a shopper collects items before committing. The cart looks simple, but it carries a marketplace-specific twist (one cart spans many vendors) and a subtle correctness point you'll lean on at checkout (the cart shows *live* prices; the order will *freeze* them). Getting these right here makes the checkout chapters that follow clean.

This is the first chapter of the commerce module (Chapters 44–48), which builds checkout one careful decision at a time: cart → split → snapshot → transaction → idempotency.

## Where we're headed

By the end a customer can add, update, and remove items in a server-side cart that may hold products from multiple vendors, view it grouped by vendor with *current* prices, all scoped to the authenticated customer.

## Step 1 — Where does the cart live?

A real decision, with three options:

| Where | Pros | Cons |
|---|---|---|
| **Client-side** (localStorage) | No server work; instant | Lost across devices; can't be used server-side at checkout |
| **Redis** (session-like) | Fast; auto-expires abandoned carts | Ephemeral; another store to reason about |
| **Database table** | Persists; multi-device; available server-side for checkout | Slightly more work than localStorage |

**The course uses a server-side database cart**, tied to the customer. The deciding reason is checkout: the *server* must read the cart to create orders (Chapter 45) and snapshot prices (Chapter 46), so the cart has to be server-side and trustworthy anyway — and persistence across devices is a real UX win. (Redis is a fine choice for a high-scale, expiry-friendly cart; the DB is the robust default here, and you already have it.)

## Step 2 — A cart is not an order

This distinction matters enormously, so be precise:

- A **cart** is **mutable** and **live**: it references products and quantities, and when you view it you show the product's *current* price. A vendor changing a price changes what the cart shows. The cart is a *wish list with quantities*, not a commitment.
- An **order** (Chapters 10, 45–47) is **immutable** and **frozen**: at checkout, prices are **snapshotted** so the order records what was actually paid, forever.

So your cart stores `product_id` + `quantity` and reads prices *live* from `products`; it does **not** snapshot. Snapshotting happens at the moment of purchase (Chapter 46), which is exactly when "what you pay" gets fixed. Confusing these — snapshotting in the cart, or reading live prices on an order — is a classic bug; keep the line sharp.

## Step 3 — The multi-vendor cart

A marketplace cart can hold a mug from Clay & Co *and* a scarf from WoolWorks at once. The cart itself doesn't care — it's just a list of `(product_id, quantity)`. But when you *display* it, group by vendor, because that's how the customer will be charged and how it'll split into orders (Chapter 45):

```
Cart (one customer)
 ├─ Clay & Co
 │    Celadon mug   × 1   @ $20 (live)
 └─ WoolWorks
      Merino scarf  × 1   @ $45 (live)
 subtotal shown per vendor and overall — all at CURRENT prices
```

This grouping previews the per-vendor order split and makes the multi-vendor nature visible to the shopper.

## Step 4 — Do it on your project (hands-on)

**1. Migrate cart tables** — a `carts` row per customer and `cart_items` (`cart_id`, `product_id` FK, `quantity`, with a `UNIQUE (cart_id, product_id)` so adding the same product updates quantity rather than duplicating). New migration (Chapter 12).

**2. Build the cart endpoints**, all guarded by `requireAuth` + `requireRole('customer')` and scoped to the caller's cart (Chapter 28 — a customer only touches *their* cart):
- `POST /cart/items` — add a product (or bump quantity); validate the product exists and is published.
- `PATCH /cart/items/:productId` — change quantity.
- `DELETE /cart/items/:productId` — remove.
- `GET /cart` — view, grouped by vendor, with **live** prices and subtotals.

**3. Test it, including the live-price point:**

```bash
C=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"buyer@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)

curl -s -X POST localhost:3000/cart/items -H "authorization: Bearer $C" -d '{"productId":"<clay mug>","quantity":1}' -H 'content-type: application/json'
curl -s -X POST localhost:3000/cart/items -H "authorization: Bearer $C" -d '{"productId":"<wool scarf>","quantity":1}' -H 'content-type: application/json'
curl -s -H "authorization: Bearer $C" localhost:3000/cart | jq '.vendors'    # → grouped by vendor, live prices

# a vendor changes the mug's price → the cart reflects it (live, not frozen)
# (PATCH the product price as the vendor, then re-GET the cart → new price shows)
```

The cart grouping by vendor and reflecting a live price change confirms both the multi-vendor shape and the not-yet-frozen behaviour.

> 💡 **Hint — validate cart contents at view *and* checkout, not just on add.** Between adding an item and checking out, a product can be unpublished, deleted, or go out of stock. Don't assume a cart item is still valid — re-check availability when viewing the cart and again at checkout, and surface "this item is no longer available." A cart is a snapshot of *intent*, not a guarantee the items are still buyable.

> **📖 Mandatory read — before Chapter 45.** Read a short piece on **cart design (server vs client carts)** and **why carts use live prices** (search *"shopping cart price at checkout vs cart"*). *Required: the cart-vs-order distinction is the foundation of the checkout chapters.*

> **Interesting to read.** "Price changed between cart and checkout" is a real e-commerce edge case every platform handles — some honour the cart price, some the checkout price, some warn. Search *"price changed at checkout ecommerce"* to see why the cart being *live* (not frozen) is a deliberate, common choice that pushes the freeze to checkout.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 45:

- [ ] A **server-side cart** (DB) is scoped to the authenticated **customer**; adding the same product **updates quantity** (unique constraint)
- [ ] A customer can **add, update, remove, and view** cart items
- [ ] The cart may hold products from **multiple vendors** and is **viewed grouped by vendor**
- [ ] Viewing the cart shows **live** prices (a vendor price change is reflected) — the cart does **not** snapshot
- [ ] Cart items are **re-validated** for availability at view/checkout
- [ ] You can explain the cart-vs-order (live vs frozen) distinction

> **✍️ Log it (mandatory).** In `learning-log/44-cart.md` — **decision** first, then **topics**: **(Decision)** Why a server-side database cart rather than client-side, given what checkout needs? Why does the cart use *live* prices instead of snapshotting? **(Topics)** (1) What's the difference between a cart and an order? (2) Why can one cart span multiple vendors, and why group by vendor when displaying? (3) Why re-validate cart items rather than trust them? (4) What `UNIQUE` constraint keeps a product from duplicating in a cart, and why?

*All boxes ticked and the log written? Then continue. The cart holds items from several makers — checkout must turn that into the right shape: one order per vendor.*

---

Next: turn one multi-vendor cart into the correct commerce structure — one order per vendor. → **[Chapter 45 — Order splitting](45-order-splitting.md)**
