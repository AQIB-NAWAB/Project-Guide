# Chapter 26 — Roles and route protection

Your authorization matrix (Chapter 25) makes the gaps obvious: `POST /products` checks that you're *authenticated* but not that you're a *vendor*, so a logged-in customer walks right in. That's **broken function-level authorization** — the first failure mode from the last chapter, and a top API risk. This chapter closes it by enforcing the *role* half of authorization: each protected route demands not just a valid token, but the *right kind* of user behind it.

It's a focused build, and it follows the now-familiar shape — a small, reusable middleware composed onto routes, exactly like `validate` (Chapter 16) and `requireAuth` (Chapter 24). And it ends on a deliberate cliffhanger: even with roles enforced, a vendor can still touch *another vendor's* product — the object-level hole that the entire next module exists to close.

## Where we're headed

By the end you'll have a `requireRole` middleware that runs after `requireAuth`, returns `403` when the authenticated user's role isn't permitted, and is composed onto routes per your matrix — so a customer hitting a vendor-only endpoint gets a clean `403`, a vendor gets through, and an anonymous request still gets `401`.

## Step 1 — Role comes from the token, never the request

Before any code, the non-negotiable from Chapter 25: the user's role is the one in their **verified access token** (attached as `req.user.role` by `requireAuth`), which came from the database at login. It is **never** read from the request body, a header the client controls, or a query param. A request that includes `{"role":"admin"}` is just noise — the mass-assignment lesson (Chapter 15) in authorization form. Your role check reads `req.user.role` and nothing else.

## Step 2 — `requireRole`: a small, composable gate

The middleware is tiny by design. It assumes `requireAuth` already ran (so `req.user` exists), and checks whether that user's role is in the allowed set:

```
// wiring — requireAuth first (who are you?), then requireRole (are you the right kind?)
router.post('/products',
  requireAuth,                       // 401 if not authenticated
  requireRole('vendor'),             // 403 if authenticated but not a vendor
  validate({ body: CreateProductBody }),
  createProductHandler)
```

On a role mismatch it returns **`403 Forbidden`** through your error contract (Chapter 17) — *not* `401`, and *not* `400`. The ordering reads like a sentence: *are you someone (401)? → are you the right kind of someone (403)? → is your input valid (400)? → do the work.* Each gate has one job.

## Step 3 — `401` vs `403`, precisely

You set up both codes in Chapter 17 and met the distinction in Chapter 24; here it becomes muscle memory:

- **No / invalid / expired token** → `requireAuth` returns **`401`** ("I don't know who you are").
- **Valid token, wrong role** → `requireRole` returns **`403`** ("I know exactly who you are; you're not allowed").

Returning the right one matters beyond tidiness: a `401` tells the client to refresh-and-retry (Chapter 22); a `403` tells it not to bother — the credentials are fine, the *permission* isn't. Conflating them sends clients chasing token refreshes that will never help.

## Step 4 — Deny by default

Apply the Chapter 25 principle as you wire routes: a route is *closed* until you deliberately open it. Public routes (browsing the catalogue) are the *explicit* exceptions; everything that writes or reveals private data carries `requireAuth` and the appropriate `requireRole`. The habit to build: when you add a new route, the first question is "who may call this?" — and the default answer, until you decide, is "no one." Forgetting a check should leave a route *locked*, not open.

## Step 5 — The hole this does *not* close (the honest cliffhanger)

Be clear-eyed about what role checks do and don't buy you. After this chapter, `POST /products` correctly demands a vendor. But consider editing a product: `PATCH /products/:id` guarded by `requireRole('vendor')` lets *any* vendor through — including one editing *another vendor's* product by putting someone else's id in the URL. The role check passed (they *are* a vendor); the **ownership** check that should ask "is this product *yours*?" doesn't exist yet.

That's **broken object-level authorization (IDOR)** — the second failure mode from Chapter 25 — and it is precisely what the next module (Chapters 27–30, multi-tenant isolation) is built to close. Role protection is necessary and you're building it now; it is not *sufficient*. Naming the remaining hole out loud is what keeps you from mistaking "has roles" for "is secure."

## Step 6 — Do it on your project (hands-on)

**1. Build `requireRole(...roles)`** — a middleware that reads `req.user.role` (set by `requireAuth`) and returns `403` via your error contract if it's not in the allowed set.

**2. Protect by your matrix.** Put `requireAuth` + `requireRole('vendor')` on `POST /products` (and other vendor-only writes); leave public reads (browsing) open; mark customer-only actions with `requireRole('customer')`.

**3. Test all three outcomes:**

```bash
# anonymous → 401 (requireAuth)
curl -i -X POST localhost:3000/products -d '{...}' -H 'content-type: application/json'                       # → 401

# logged-in CUSTOMER → 403 (requireRole) — the function-level hole, now closed
CUST=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"buyer@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)
curl -i -X POST localhost:3000/products -H "authorization: Bearer $CUST" -d '{...}'                          # → 403

# logged-in VENDOR → 201
VEND=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"maker@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)
curl -i -X POST localhost:3000/products -H "authorization: Bearer $VEND" -d '{"title":"Mug","price_cents":2000,"currency":"USD","status":"draft"}' -H 'content-type: application/json'   # → 201
```

Customer → `403`, vendor → `201`, anonymous → `401`: that's function-level authorization working. Tick the row in your `AUTHORIZATION.md` matrix.

**4. Prove the role can't be faked.** Send a customer's token *with* `{"role":"vendor"}` in the body and confirm it's still `403` — the body is ignored; the role comes from the token.

> 💡 **Hint — order the gates deliberately, and keep them thin.** `requireAuth` before `requireRole` before `validate` before the handler. If a customer somehow reaches a vendor handler, check the route actually has `requireRole` attached and in the right place. Resist baking role logic *into* handlers — keep it in the composable middleware so every route reads its protection at a glance and you can audit them at once.

> **📖 Mandatory read — before Chapter 27.** Re-read the **OWASP "Broken Function Level Authorization"** entry now that you've fixed it, and skim **"Broken Object Level Authorization"** — *Required: the next module is dedicated entirely to that second one, multi-tenant isolation.*

> **Interesting to read.** Plenty of breaches come from *function-level* gaps specifically — an admin or vendor-only endpoint that only checked authentication, letting any logged-in user call privileged actions just by knowing the URL. Search *"broken function level authorization example"* to see how often "logged in" was mistaken for "allowed."

## Definition of Done

Things you can **see or run** — and the gate to Chapter 27:

- [ ] A **`requireRole` middleware** reads the role from `req.user` (the verified token) and returns **`403`** on a mismatch
- [ ] Role comes **only** from the token — sending `{"role":"vendor"}` in the body does **not** grant access (you tested it)
- [ ] `POST /products` (and other vendor-only routes) require **vendor**; a logged-in **customer gets `403`**, a **vendor gets through**, anonymous gets **`401`** — all verified
- [ ] You apply **deny-by-default**: routes are protected unless explicitly public
- [ ] You can state why role protection is **necessary but not sufficient**, and name the object-level hole still open

> **✍️ Log it (mandatory).** In `learning-log/26-roles-and-route-protection.md` — **decision** first, then **topics**: **(Decision)** Why must the role be read from the verified token rather than the request, and what would break if you trusted the body? Why is `requireRole` a separate middleware rather than an `if` inside each handler? **(Topics)** (1) Walk the exact `401` vs `403` outcomes for anonymous, customer, and vendor on `POST /products`. (2) What is "deny by default," and how does it make a forgotten check fail safe? (3) Why does a passing role check still leave an IDOR hole on `PATCH /products/:id`? (4) What does "necessary but not sufficient" mean for role-based authorization here?

*All boxes ticked and the log written? Then continue. Roles now gate every action by *kind* of user — but a vendor can still reach another vendor's *specific* data. The next module makes the marketplace truly multi-tenant: your data is yours, and no one else's.*

---

Next: roles stop the wrong *kind* of user — but not the right kind touching the wrong *data*. Begin multi-tenant isolation, the heart of a real marketplace. → **[Chapter 27 — Multi-tenant isolation: the concept](27-isolation-concept.md)**
