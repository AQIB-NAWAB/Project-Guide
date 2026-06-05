# Chapter 47 — The checkout transaction

Your checkout now does a lot: it creates several orders, many order items with snapshotted prices, and clears the cart (Chapters 45–46). That's a series of separate database writes — and here's the danger: what if it fails *halfway*? The server crashes, the connection drops, or one write errors after others succeeded. You're left with a corrupt in-between state: orders created but the cart not cleared, or some order items missing, or one vendor's order written and the other's lost. A half-finished checkout is worse than a failed one, because the data *looks* real but is wrong.

The fix is **transactions** — the database's mechanism for "all of these writes succeed together, or none of them happen." This is the **ACID** guarantee you first met way back in Chapter 2, finally put to work at the moment it matters most.

## Where we're headed

By the end the entire checkout runs inside one database transaction — all the orders, items, and the cart clear commit together or roll back together — and you've proven that a failure mid-checkout leaves *no* partial data.

## Step 1 — The corruption a missing transaction causes

Picture checkout without a transaction, failing after the first order is written:

```
✓ create Clay & Co order + items
✓ snapshot prices
✗ create WoolWorks order  ← fails here (crash / error)
   cart not cleared, WoolWorks order missing
```

Now the database holds a Clay & Co order for a purchase that *partially* happened. The customer was maybe charged, the cart still shows items, one vendor has an order and the other doesn't. There's no clean way to recover, because you can't tell a "real" order from a "half-checkout" order. Every multi-write operation has this failure mode, and checkout — touching money — is the worst place for it.

## Step 2 — Atomicity: all or nothing

A **transaction** wraps multiple statements so they behave as a single, indivisible unit. The shape (you saw this in Chapter 2):

```sql
BEGIN;
  -- create all orders
  -- create all order_items (with snapshots)
  -- clear the cart
COMMIT;     -- everything becomes real, together
-- if ANYTHING fails before COMMIT:
ROLLBACK;   -- the database is left EXACTLY as it was before BEGIN
```

This is the **A** in ACID — **atomicity**: the transaction either fully commits or fully rolls back, never in between. Wrap the whole checkout in one, and the half-finished state from Step 1 becomes *impossible* — a failure rolls back every write, leaving the cart intact and no orphan orders. The customer can simply retry.

## Step 3 — What ACID buys you (briefly)

While you're here, the rest of ACID matters to checkout too:

- **Atomicity** — all-or-nothing (the headline above).
- **Consistency** — the transaction moves the database from one valid state to another; your constraints (Chapter 11) hold throughout, so a transaction that would violate them fails as a unit.
- **Isolation** — concurrent transactions don't see each other's half-done work (relevant if two checkouts run at once; isolation levels tune this).
- **Durability** — once committed, it survives a crash.

For checkout, atomicity is the star, but consistency (your constraints enforced even mid-transaction) and isolation (concurrent checkouts not corrupting each other) are quietly doing important work too.

## Step 4 — Keep the transaction tight

One practical rule: a transaction holds locks and resources while open, so keep it **short and focused** — only the writes that must be atomic. *Don't* do slow, external work inside it (sending a confirmation email, charging a payment provider, generating a PDF) — those make the transaction long, hold locks, and can fail in ways a rollback can't undo (you can't un-send an email). The pattern: do the fast, atomic database work in the transaction; push the slow/external work *outside* it — onto the background queue you'll build in Week 4 (the confirmation email becomes a job, Chapter 51). Note that seam now.

## Step 5 — Do it on your project (hands-on)

**1. Wrap checkout in a transaction.** Put the entire Chapter 45–46 checkout — create orders, create snapshotted order items, clear the cart — inside one database transaction (your ORM/query layer's transaction API). Commit at the end; roll back on any error.

**2. Prove atomicity by forcing a failure:**

```bash
# temporarily make the LAST step of checkout throw (e.g. after creating the first order)
# then attempt checkout:
curl -i -X POST localhost:3000/checkout -H "authorization: Bearer $C"      # → 500 (it failed)

# verify NOTHING was created and the cart is intact
psql "$DATABASE_URL" -c "SELECT count(*) FROM orders WHERE checkout_id='<would-be-id>';"   # → 0  (rolled back!)
curl -s -H "authorization: Bearer $C" localhost:3000/cart | jq '.vendors | length'         # → cart still full

# remove the forced failure → checkout succeeds, everything commits together
curl -i -X POST localhost:3000/checkout -H "authorization: Bearer $C"      # → 201, cart cleared
```

Forcing a mid-checkout failure and confirming **zero** partial data — no orphan orders, cart untouched — is atomicity proven. Then the clean run commits everything at once.

**3. Add a test** asserting a failed checkout leaves no orders and a full cart (regression guard, like Chapters 30 and 41).

> 💡 **Hint — the transaction must wrap the *write*, not the read.** Validate the cart and check availability *before* `BEGIN` where you can, but the actual order/item creation and cart clear must all be inside one transaction. And confirm your cache invalidation (Chapter 41) fires *after* commit, not inside the transaction — invalidating before commit can re-cache pre-checkout state.

> **📖 Mandatory read — before Chapter 48.** Read about **database transactions and ACID** (search *"database transaction ACID explained"*) and your ORM's **transaction API**. *Required: Chapter 48's idempotency check lives inside this transaction.*

> **Interesting to read.** "We processed the order but the payment/cart/inventory got out of sync" is one of the most damaging classes of e-commerce bug, and it's almost always a missing or mis-scoped transaction. Search *"order processing partial failure transaction"* to see why wrapping checkout atomically is non-negotiable.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 48:

- [ ] The **entire checkout** (orders + items + cart clear) runs in **one transaction**
- [ ] A failure mid-checkout **rolls back everything** — you forced one and confirmed **zero** partial data and an intact cart
- [ ] A successful checkout **commits everything together**
- [ ] Slow/external work (confirmation email) is **kept outside** the transaction (seam noted for Week 4)
- [ ] A **test** asserts a failed checkout leaves no orders and a full cart
- [ ] You can explain atomicity and why a half-checkout is worse than a failed one

> **✍️ Log it (mandatory).** In `learning-log/47-checkout-transaction.md` — **decision** first, then **topics**: **(Decision)** Why must checkout be wrapped in a single transaction, and why keep external work (email, payment) *outside* it? **(Topics)** (1) Describe the corrupt state a missing transaction allows. (2) What does each letter of ACID guarantee, and which matters most for checkout? (3) Why keep transactions short and free of slow external calls? (4) What did forcing a failure prove about your checkout?

*All boxes ticked and the log written? Then continue. Checkout is atomic — but a customer clicking "pay" twice, or a retry, could still create two orders. Make it safe to submit more than once.*

---

Next: a double-click or network retry must not create two orders — make checkout idempotent. → **[Chapter 48 — Idempotent checkout](48-idempotent-checkout.md)**
