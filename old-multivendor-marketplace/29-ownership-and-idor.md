# Chapter 29 — Ownership and IDOR

Query scoping (Chapter 28) made other tenants' rows invisible to *reads*. This chapter closes the matching hole on *writes* — the exact one you've been flagging since Chapter 26: a vendor editing or deleting **another vendor's** product by putting its id in the URL. That's **IDOR** (insecure direct object reference), the #1 API risk, and after this chapter it's shut.

The fix is the second isolation layer: an **explicit ownership check** on every operation that targets a specific object by a client-supplied id. It's a small amount of code with an outsized security payoff — and the satisfying part is that it finally puts the entire Chapter 8 ownership chain to work defending real mutations.

## Where we're headed

By the end, `PATCH /products/:id` and `DELETE /products/:id` confirm the product belongs to the caller's store before doing anything; a vendor acting on another vendor's product gets a clean `404`; and you've closed the IDOR cliffhanger from Chapter 26.

## Step 1 — The vulnerable handler, in full

Here's the IDOR bug concretely. A natural-looking edit handler:

```
// ❌ IDOR — checks you're a vendor, never checks the product is YOURS
PATCH /products/:id   (requireAuth + requireRole('vendor'))
  load product by :id            // any product, anyone's
  apply the updates
  save
```

Every gate *looks* present — authenticated, correct role, valid body. But `requireRole('vendor')` only asks "are you *a* vendor?" — and the attacker *is* one. They pass their own token, put a *different* vendor's product id in the URL, and edit (or delete) it. Roles guarded the action; nothing guarded the *object*. This is the confused deputy (Chapter 27): your trusted server acting on B's data on A's behalf.

## Step 2 — The fix: verify ownership before acting

The rule is absolute: **never act on an object identified by a client-supplied id without confirming it belongs to the current tenant.** Two equivalent ways to enforce it:

- **Scoped mutation** (preferred — reuse Chapter 28): the `UPDATE`/`DELETE` itself carries the tenant filter, so it only affects the row if it's yours:

```sql
-- ✅ the update touches the row ONLY if it's the caller's
UPDATE products SET ... WHERE id = $1 AND store_id = $2;   -- $2 = caller's store
-- affected rows = 0  → it wasn't yours → respond 404
```

- **Load-then-check**: fetch the product *scoped* to the caller's store first; if nothing comes back, it isn't theirs — stop with `404` before any write.

Either way, the ownership check and the action are bound together so there's no gap between them. The scoped-mutation form is the most robust because a row that isn't yours simply *can't* be affected — there's no separate check to forget.

## Step 3 — Return `404`, not `403` (the deliberate choice)

When a vendor targets another's product, which is right — `403 Forbidden` ("this exists but isn't yours") or `404 Not Found` ("there's nothing here")? Both are defensible, but for cross-tenant access the course mandates **`404`**, because `403` *confirms the object exists*, which is an information leak: an attacker probing ids could map out which ids are real across the whole marketplace. `404` reveals nothing — to vendor A, vendor B's product simply does not exist. (Reserve `403` for cases where the user legitimately *knows* the object exists but lacks permission — e.g. the unverified-email gate in Chapter 23, where hiding existence isn't the point.)

This pairs perfectly with scoping (Chapter 28): a scoped query returns no row → your handler's normal "not found → `404`" path fires automatically. Isolation and your error contract do the work together.

## Step 4 — Do it on your project (hands-on)

**1. Build `PATCH /products/:id`** (edit) and **`DELETE /products/:id`** through the scoped accessor from Chapter 28, so both only affect a row whose `store_id` is the caller's. Validate the `:id` (UUID) and the body (Chapter 16); guard with `requireAuth` + `requireRole('vendor')`; on zero rows affected, respond `404`.

**2. Prove IDOR is closed with two tenants:**

```bash
A=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"a@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)
BPID=<a product id owned by vendor B>

# A edits B's product → blocked as not found
curl -i -X PATCH localhost:3000/products/$BPID -H "authorization: Bearer $A" -d '{"title":"hijacked"}' -H 'content-type: application/json'   # → 404

# A deletes B's product → blocked
curl -i -X DELETE localhost:3000/products/$BPID -H "authorization: Bearer $A"                                                                # → 404

# A edits A's OWN product → works
APID=<a product id owned by vendor A>
curl -i -X PATCH localhost:3000/products/$APID -H "authorization: Bearer $A" -d '{"title":"new name"}' -H 'content-type: application/json'    # → 200
```

A cannot touch B's product (it "doesn't exist" to them), but can edit their own. The Chapter 26 cliffhanger is resolved.

**3. Confirm the snapshot still protects history.** Edit a product's price, then check an order that bought it (Chapters 10–11): the order is unchanged. Ownership controls who can edit the *live* product; snapshots keep *past sales* immune regardless.

**4. Tick `ISOLATION.md` and `AUTHORIZATION.md`** — the product edit/delete rows now have ownership enforcement.

> 💡 **Hint — check ownership for *every* object id, not just products.** The same rule applies the moment any endpoint takes an id from the client: an order id, a store id, later an image id. Build the habit now — *any* client-supplied id that names a vendor-owned object must be combined with the tenant scope before you read or write it. The next chapter will try to break exactly this across your API.

> **📖 Mandatory read — before Chapter 30.** Re-read **OWASP "Broken Object Level Authorization"** with your fix in hand, and read a short piece on **403 vs 404 for authorization** (search *"return 404 instead of 403 security"*). *Required: Chapter 30 writes automated tests asserting these exact behaviours.*

> **Interesting to read.** IDOR is behind a remarkable number of real breaches precisely because the fix is *so small* it's easy to skip — a four-line ownership check absent from one endpoint has exposed millions of records (medical, financial, personal). Search *"IDOR vulnerability real world example"* to see how a missing `AND owner_id = me` becomes a disclosure incident.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 30:

- [ ] `PATCH /products/:id` and `DELETE /products/:id` only affect a product whose `store_id` is the **caller's** (scoped mutation or load-then-check)
- [ ] Vendor A editing or deleting vendor B's product returns **`404`** (you tested both) — never `403` and never success
- [ ] A vendor can edit and delete their **own** products
- [ ] You can justify the **`404`-not-`403`** choice for cross-tenant access
- [ ] The IDOR hole flagged in Chapter 26 is **closed**, and `ISOLATION.md`/`AUTHORIZATION.md` updated

> **✍️ Log it (mandatory).** In `learning-log/29-ownership-and-idor.md` — **decision** first, then **topics**: **(Decision)** Why is an explicit ownership check required even though `requireRole('vendor')` already passed — what does each check actually verify? Why return `404` rather than `403` for another tenant's object? **(Topics)** (1) Show the vulnerable vs fixed mutation and explain exactly what the fix changes. (2) Why is the *scoped mutation* form more robust than a separate ownership check? (3) Why does returning `403` leak information that `404` doesn't? (4) Which other endpoints in your app take a client-supplied object id, and what must each one do?

*All boxes ticked and the log written? Then continue. Isolation works when you test it by hand — but a security property must never silently regress. Next you lock it in with an automated attack test.*

---

Next: you proved isolation by hand — now make it a test that fails the build the day someone breaks it. → **[Chapter 30 — The isolation attack test](30-isolation-attack-test.md)**
