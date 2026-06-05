# Chapter 48 — Idempotent checkout

Your checkout is atomic — it fully succeeds or fully rolls back (Chapter 47). But there's one more way it can go wrong, and it's extremely common in the real world: the customer clicks "Pay" twice, or their phone loses signal and the app retries, or a flaky network makes the request arrive twice. Each arrival is a *valid, complete* checkout — so each one creates orders. The customer wanted to buy once and got charged twice, with duplicate orders. Atomicity doesn't help here: both transactions succeed perfectly; the problem is that there were *two* of them.

The fix is **idempotency** — making it safe to submit the same operation more than once, so the second (and third) attempt has no additional effect. It's a cornerstone of reliable systems, and it's the capstone of your commerce module.

## Where we're headed

By the end, checkout is idempotent: a client sends an **idempotency key**, and a repeat request with the same key returns the **original** result instead of creating duplicate orders — proven by submitting the same checkout twice and getting one set of orders.

## Step 1 — The double-submit problem

Picture it concretely:

```
click "Pay" → POST /checkout → creates 2 orders, charges $65   ✓
(network hiccup, app auto-retries the same request)
            → POST /checkout → creates 2 MORE orders, charges $65 again   ✗✗
```

Both requests were legitimate and both succeeded. The customer now has four orders and a double charge. This isn't a rare edge case — double-clicks, mobile network retries, and impatient users are *constant*, which is why every payment and checkout system in the world handles it.

## Step 2 — What idempotency means

An operation is **idempotent** if performing it many times has the **same effect as performing it once**. `GET` and `DELETE` are naturally idempotent (reading twice, or deleting an already-deleted thing, changes nothing more). `POST /checkout` is *not* — each call creates new orders. You make it idempotent with an **idempotency key**:

- The client generates a **unique key** for *one* checkout attempt (a UUID) and sends it (e.g. an `Idempotency-Key` header).
- The server, the first time it sees that key, processes the checkout and **records the key with its result**.
- If a request arrives with a key it has **already seen**, the server **does not re-process** — it returns the **stored result** of the first attempt.

So the retry gets the *same* "here are your 2 orders" response the original did, but no new orders are created. The client can safely retry as many times as it likes.

## Step 3 — Making the check race-safe

The subtle part: two identical requests can arrive *simultaneously* (a genuine double-fire), both check "have I seen this key?", both see "no," and both proceed — the exact duplicate you're preventing. So the key check must be **atomic** with the work. Two robust approaches:

- **A unique constraint** on the idempotency key in a table: insert the key as the *first* step inside the checkout transaction (Chapter 47). If a concurrent request already inserted it, the `UNIQUE` constraint rejects the second — and that transaction rolls back, returning the first's result. The database arbitrates the race for you.
- **An atomic check-and-set in Redis** (`SET key ... NX`) before processing.

Putting the key insert *inside* the Chapter 47 transaction is clean: the key and the orders commit together, so "key recorded" and "orders created" are inseparable — you can never record the key but fail the orders, or vice versa.

## Step 4 — Do it on your project (hands-on)

**1. Add an idempotency store** — an `idempotency_keys` table (`key UNIQUE`, `customer_id`, the stored response/order ids, `created_at`) via a new migration (Chapter 12).

**2. Accept and enforce the key.** `POST /checkout` requires an `Idempotency-Key`. Inside the checkout transaction (Chapter 47), insert the key *first*; on a duplicate-key conflict, abort and return the original result instead of creating new orders.

**3. Prove no duplicates:**

```bash
KEY=$(uuidgen)

# first submit → creates orders
curl -s -X POST localhost:3000/checkout -H "authorization: Bearer $C" -H "idempotency-key: $KEY" | jq '.checkoutId'   # → checkout A

# SAME key again (simulating a retry/double-click) → returns the SAME result, no new orders
curl -s -X POST localhost:3000/checkout -H "authorization: Bearer $C" -H "idempotency-key: $KEY" | jq '.checkoutId'   # → checkout A (identical)

# confirm only ONE set of orders exists
psql "$DATABASE_URL" -c "SELECT count(DISTINCT checkout_id) FROM orders WHERE customer_id='<id>';"   # → 1, not 2

# a DIFFERENT key → a new checkout (a genuine second purchase)
curl -s -X POST localhost:3000/checkout -H "authorization: Bearer $C" -H "idempotency-key: $(uuidgen)"   # → checkout B
```

Same key → same single result; different key → new order. That's idempotency: safe to retry, impossible to double-charge by accident.

**4. Add a test** firing the same checkout twice with one key and asserting exactly one set of orders — a regression guard for a property you never want to lose.

> 💡 **Hint — idempotency is about to matter for *workers* too.** You're building this for HTTP double-submits, but the same idea returns in Week 4: background jobs can be delivered more than once (Chapter 52), so a job that sends an email or charges a card must be idempotent for the *same* reason a checkout must be. The mental model — "make repeat execution a no-op" — transfers directly. You're learning a principle, not just a checkout trick.

> **📖 Mandatory read — before Chapter 49.** Read **Stripe's guide to idempotent requests** (search *"stripe idempotency keys"*) — the canonical explanation, from a payments company that lives by it. *Required: idempotency recurs throughout Week 4's async work.*

> **Interesting to read.** Payment APIs make idempotency keys *mandatory* on charge requests precisely because networks retry and users double-click — it's the difference between "retry is safe" and "retry double-charges the customer." Search *"idempotency key payment API"* to see why this small mechanism underpins financial reliability.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 49:

- [ ] `POST /checkout` requires an **idempotency key** and stores it with the result
- [ ] A repeat request with the **same key** returns the **original result** and creates **no new orders** (you verified)
- [ ] A request with a **different key** creates a new checkout
- [ ] The key check is **race-safe** (unique constraint inside the transaction, or atomic Redis set)
- [ ] A **test** asserts a doubled request produces exactly one set of orders
- [ ] You can explain what idempotency means and why atomicity alone didn't prevent double-submits

> **✍️ Log it (mandatory).** In `learning-log/48-idempotent-checkout.md` — **decision** first, then **topics**: **(Decision)** Why does checkout need idempotency on top of being atomic — what does each prevent? Why put the idempotency-key insert *inside* the checkout transaction? **(Topics)** (1) Define idempotency and give a naturally-idempotent HTTP method. (2) How does an idempotency key make `POST /checkout` safe to retry? (3) How do you make the key check race-safe against simultaneous requests? (4) Where else (Week 4) will you need this same idea?

*All boxes ticked and the log written? **That completes the commerce module and Week 3.** Customers can browse, search, cart, and check out — atomically, idempotently, with honest frozen prices. Week 4 makes the system resilient: moving slow work off the request, then hardening and shipping.*

---

Next: checkout works — but it still does slow things (emails, image processing) inline. Week 4 begins by moving that work off the request path. → **[Chapter 49 — Why async?](49-why-async.md)**
