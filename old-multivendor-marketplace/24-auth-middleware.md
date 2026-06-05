# Chapter 24 — Authentication middleware

You issue access tokens on login (Chapter 21), refresh them (Chapter 22), and verify emails (Chapter 23) — but step back and notice something: *nothing actually checks the token on an incoming request yet.* `POST /products` is still wide open; a request with no token, or a forged one, sails straight through. You have all the pieces of authentication except the one that does the *authenticating* on each request.

This chapter builds that gate — an **authentication middleware** that reads and verifies the access token on protected routes and attaches the caller's identity for everything downstream. It's the exact parallel of validation: you understood the token by hand in Chapter 21; now you turn it into a reusable middleware applied across routes (just as Chapter 16 generalised Chapter 15). And it pays off a debt — that `// TODO: derive store from auth` you left in Chapter 15 finally gets resolved.

## Where we're headed

By the end you'll have a `requireAuth` middleware that verifies the `Bearer` access token, rejects missing/invalid/expired tokens with `401`, and attaches `req.user` (id + role) for handlers; `POST /products` will be **protected** and will derive the vendor's store from the authenticated user instead of trusting the request body.

## Step 1 — Why a middleware, not a check in every handler

You *could* verify the token at the top of every protected handler. You already know where that goes — it's the Chapter 16 rot all over again: duplicated verification logic, inconsistent `401`s, and the one handler that forgets and becomes an open door. Authentication that lives *inside* each handler is authentication each handler can skip.

So lift it into one **middleware** that runs *before* the handler. Guarding a route becomes one visible, declarative step, and by the time a protected handler runs, the caller is guaranteed authenticated and their identity is available.

## Step 2 — What `requireAuth` does

The middleware does a tight, well-defined job on every protected request:

1. **Read the token** from the `Authorization: Bearer <token>` header. Missing or malformed header → `401`.
2. **Verify it** — check the signature against `JWT_SECRET` (Chapter 21) *and* that it hasn't expired. Verify **strictly**: pin the expected algorithm (never accept `alg: none` — Chapter 21's interesting read), and reject anything that doesn't validate → `401`.
3. **Attach identity.** On success, decode the claims and attach `req.user = { id, role }` so handlers and later authorization checks can use it — *without* a database lookup, because the verified token carries them.

```
// wiring — requireAuth runs before the handler; you write the middleware body
router.post('/products', requireAuth, validate({ body: CreateProductBody }), createProductHandler)
//                       └─ no/invalid/expired token → 401, handler never runs
```

The handler downstream can now simply trust `req.user`. (Whether that user is *allowed* to do the action — role, ownership — is authorization, the next module. `requireAuth` only answers *who*.)

## Step 3 — `401` vs `403`: get the distinction right

This trips people up, and your error contract (Chapter 17) already has both codes for a reason:

- **`401 Unauthorized`** = *not authenticated.* No token, an invalid token, an expired one. The system doesn't know who you are. (When the access token is expired, this `401` is the client's cue to hit `/auth/refresh` — Chapter 22 — and retry.)
- **`403 Forbidden`** = *authenticated, but not allowed.* We know exactly who you are; you just can't do this. (That's the next module — roles and ownership.)

`requireAuth` only ever produces `401`. A `403` means a *different* gate (authorization) rejected a perfectly authenticated user. Keeping them distinct lets clients react correctly — refresh-and-retry on `401`, show "not permitted" on `403`.

## Step 4 — Protect routes, and pay off the Chapter 15 debt

Now apply it — and watch an old loose end tie up. Back in Chapter 15 you built `POST /products` with a note: `// TODO: derive store from auth`, and you temporarily let the request *say* which store (a mass-assignment risk you flagged in Chapter 15). With `requireAuth`, the server now *knows* the caller: `req.user.id` is the authenticated vendor, so you look up **their** store and attach the product to it. The client no longer sends `store_id` at all — there's nothing to forge.

```
// before (Ch15): store came from the untrusted body  ❌
// after  (Ch24): store derived from the verified token ✅
POST /products  (with Bearer token)
  → server uses req.user.id → that vendor's store → creates the product there
```

This is the moment authentication *earns its keep*: the identity in the token replaces a field the client used to control, closing the mass-assignment hole for real. Remove `store_id` from the accepted body schema.

## Step 5 — Do it on your project (hands-on)

**1. Build `requireAuth`.** One middleware: read the `Bearer` header, verify the JWT strictly with `JWT_SECRET`, `401` on any failure, attach `req.user = { id, role }` on success.

**2. Protect `POST /products`** by putting `requireAuth` in front of it.

**3. Derive the store from `req.user`.** Update the handler to look up the authenticated vendor's store and use it; **delete `store_id` from the request body schema**. The Chapter 15 TODO is now done.

**4. Test the gate:**

```bash
# no token → 401
curl -i -X POST localhost:3000/products -d '{...}' -H 'content-type: application/json'                # → 401

# garbage / tampered token → 401
curl -i -X POST localhost:3000/products -H 'authorization: Bearer not.a.token' -d '{...}'             # → 401

# valid token (from login) → works, product created in THAT vendor's store
TOKEN=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"m@x.com","password":"a-long-passphrase"}' -H 'content-type: application/json' | jq -r .accessToken)
curl -i -X POST localhost:3000/products -H "authorization: Bearer $TOKEN" -d '{"title":"Celadon mug","price_cents":2000,"currency":"USD","status":"draft"}' -H 'content-type: application/json'   # → 201
```

No token and a bad token both return `401`; a valid token creates the product **in the authenticated vendor's store**, with no `store_id` in the request. That's authentication, enforced.

**5. (Optional) An `optionalAuth` variant** for routes that behave differently logged-in vs out (a public catalogue that shows extra info to owners) — same verification, but a missing token is allowed and simply leaves `req.user` undefined.

> 💡 **Hint — middleware order, again.** `requireAuth` must run *before* the handler (so it can block) and, where both apply, think about its order relative to `validate`: typically authenticate first (is this a known caller at all?), then validate the body. If a protected route still lets anonymous requests through, check that the middleware is actually attached to that route and runs before the handler — the same ordering question you met in Chapter 16.

> **📖 Mandatory read — before Chapter 25.** Read your framework's docs on **writing authentication middleware** and your JWT library's **strict verification** options (algorithm pinning, expiry). *Required: Chapter 25 begins authorization, which runs immediately after this gate and depends on the `req.user` it attaches.*

> **Interesting to read.** Many real API breaches aren't broken crypto — they're a protected route that simply *forgot* its auth middleware, or verified the token loosely (accepting `alg: none`). Search *"broken authentication API top 10"* (OWASP API Security) to see why a single consistent gate, applied everywhere and verifying strictly, is the actual defence.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 25:

- [ ] A **`requireAuth` middleware** reads the `Bearer` token, **verifies it strictly** (signature + expiry, algorithm pinned), and attaches `req.user`
- [ ] A request with **no token or an invalid/expired token returns `401`** (you tested both)
- [ ] A request with a **valid token succeeds**, and the handler can read `req.user`
- [ ] `POST /products` is **protected** and now **derives the store from `req.user`** — `store_id` is no longer accepted from the body (the Chapter 15 TODO is closed)
- [ ] You can state precisely when to return **`401` vs `403`**

> **✍️ Log it (mandatory).** In `learning-log/24-auth-middleware.md` — **decision** first, then **topics**: **(Decision)** Why verify the token in one middleware instead of in each protected handler? Why is deriving the store from `req.user` more secure than accepting `store_id` in the body? **(Topics)** (1) What three things does `requireAuth` do on each request, and why does it need no database lookup? (2) Explain `401` vs `403` and what a client should do on each. (3) Why must token verification be *strict* (algorithm pinned, expiry enforced)? (4) How does this chapter finally close the mass-assignment risk you flagged back in Chapter 15?

*All boxes ticked and the log written? Then continue. Your app now knows **who** is calling every protected route — which immediately raises the next question: who is allowed to do **what**? That's authorization, and it starts now.*

---

Next: you know who's calling — but a customer can still hit a vendor-only endpoint, and a vendor can still touch another vendor's data. Time to decide who's allowed to do what. → **[Chapter 25 — Authorization: who can do what](25-authz-concept.md)**
