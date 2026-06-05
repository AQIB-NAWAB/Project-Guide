# Chapter 31 — Vendor onboarding

The marketplace is now secure: vendors authenticate, roles gate actions, and isolation seals each tenant's data (Chapters 18–30). But a freshly registered vendor is in an odd, incomplete state — they have an account and a role, yet **no store**. They can't list a product because there's nowhere to put it. Onboarding is the step that turns "a person with a vendor account" into "a maker with an open shop," and it's where several threads you've built finally come together: email verification, the one-store-per-vendor rule, and tenant-scoped creation.

This is a build chapter, and a satisfying one — it composes auth, roles, verification, and isolation into a single real workflow, and it's the gateway every later vendor feature depends on.

## Where we're headed

By the end a verified vendor can open exactly one store (name, unique slug, description) via a guarded endpoint; a second attempt is rejected; an unverified vendor is blocked; and the store is created tied to the authenticated vendor.

## Step 1 — The onboarding state

Recall the model: registration (Chapter 20) created the `accounts` row and the `vendor_profiles` row, but Chapter 8 mandated **one store per vendor** as a *separate* `stores` table with a `UNIQUE` `vendor_id`. So a new vendor exists *without* a store until they onboard. Your app should recognise this state — "vendor, no store yet" — and guide them to create one. (Your frontend in Chapter 34 will route a storeless vendor to onboarding.)

Opening the store is the act of becoming a real seller, so it's the natural place to enforce the trust requirements you built.

## Step 2 — Compose the guards you already have

Store creation is where several earlier gates stack, in order:

- **`requireAuth`** (Chapter 24) — must be logged in.
- **`requireRole('vendor')`** (Chapter 26) — only vendors open stores.
- **Verified email** (Chapter 23) — a store is public, trust-bearing presence, so require `email_verified`; otherwise `403`. This is exactly the kind of action the verification gate was built for.

```
POST /vendor/store   (requireAuth → requireRole('vendor') → requireVerified)
  { "name": "Clay & Co", "description": "Hand-thrown stoneware" }
  → 201  the new store, tied to req.user
```

The vendor never sends their `vendor_id` — it's derived from `req.user` (Chapters 24, 28). The client states *what* the store is; the server decides *whose* it is.

## Step 3 — Enforce one store per vendor

The `UNIQUE` constraint on `stores.vendor_id` (Chapter 8) is your backstop, but as with registration (Chapter 20), check first and return a clean error rather than letting the constraint crash: if this vendor already has a store, respond **`409 Conflict`** ("you already have a store"). One maker, one shop — decided in Chapter 8, enforced here.

## Step 4 — The slug: a unique, URL-safe handle

A store needs a **slug** — the URL-safe identifier in `/stores/clay-and-co` (Chapter 8 gave `stores.slug` a `UNIQUE`). Two decisions:

- **Generate or accept?** Derive a slug from the store name (lowercase, hyphenate, strip punctuation: "Clay & Co" → `clay-and-co`), and let the vendor optionally customise it. Generating a sensible default is friendlier than demanding they invent one.
- **Uniqueness.** Slugs must be globally unique (they're public URLs). On a collision, either append a discriminator (`clay-and-co-2`) or ask the vendor to choose another with a `409`. Decide one approach and apply it consistently.

A slug is user-facing and permanent-ish (links depend on it), so validate it (Chapter 16): lowercase, hyphens, a length bound, no reserved words.

## Step 5 — Do it on your project (hands-on)

**1. Build `POST /vendor/store`**, guarded by `requireAuth` + `requireRole('vendor')` + your verified-email check. Validate the body (name, optional slug, description).

**2. Derive ownership and enforce one-per-vendor:** tie the store to `req.user`; if the vendor already has one, return `409`.

**3. Generate and uniquify the slug** from the name, validate it, and handle collisions consistently.

**4. Walk the onboarding flow:**

```bash
V=$(curl -s -X POST localhost:3000/auth/login -d '{"email":"new-vendor@x.com","password":"..."}' -H 'content-type: application/json' | jq -r .accessToken)

# unverified vendor → blocked
curl -i -X POST localhost:3000/vendor/store -H "authorization: Bearer $V" -d '{"name":"Clay & Co"}' -H 'content-type: application/json'   # → 403 (verify email first)

# after verifying (Chapter 23 flow) → store opens
curl -i -X POST localhost:3000/vendor/store -H "authorization: Bearer $V" -d '{"name":"Clay & Co"}' -H 'content-type: application/json'   # → 201, slug "clay-and-co"

# second store attempt → blocked
curl -i -X POST localhost:3000/vendor/store -H "authorization: Bearer $V" -d '{"name":"Second Shop"}' -H 'content-type: application/json'  # → 409 (already have a store)
```

Blocked-while-unverified → opens-when-verified → one-only: that arc is onboarding working, with every guard from the auth module composing cleanly.

> 💡 **Hint — a product needs a store; handle the storeless vendor.** Your `POST /products` (Chapter 24) derives the vendor's store — but a vendor who hasn't onboarded *has* no store. Decide the behaviour now: trying to create a product without a store should return a clear error ("open your store first"), not crash on a missing store lookup. Onboarding is a prerequisite for the product features that follow.

> **📖 Mandatory read — before Chapter 32.** Read a short piece on **slug generation and uniqueness** (search *"url slug generation best practices"*). *Required: slugs are public, permanent-ish identifiers and getting uniqueness right matters for the catalogue.*

> **Interesting to read.** Marketplaces treat onboarding as a funnel they obsess over — every extra step loses sellers, but too few invites fraud and low-quality shops. The verification-gated, one-shop-per-maker flow you built is the balance most handmade-goods marketplaces strike. Search *"marketplace seller onboarding friction"*.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 32:

- [ ] `POST /vendor/store` is guarded by **auth + vendor role + verified email**; an unverified vendor gets **`403`**
- [ ] The store is tied to **`req.user`** (vendor id derived from the token, never the body)
- [ ] A vendor with an existing store gets **`409`** on a second attempt — one store per vendor
- [ ] A **unique, URL-safe slug** is generated from the name, validated, and collisions handled
- [ ] A storeless vendor attempting to create a product gets a **clear error**, not a crash

> **✍️ Log it (mandatory).** In `learning-log/31-vendor-onboarding.md` — **decision** first, then **topics**: **(Decision)** Why require a *verified* email to open a store, and why enforce one store per vendor in code as well as the DB constraint? **(Topics)** (1) Which guards compose on store creation, and in what order? (2) Why is the store tied to `req.user` rather than a `vendor_id` in the body? (3) What makes a good slug, and how do you keep slugs unique? (4) How should the app treat a vendor who has no store yet?

*All boxes ticked and the log written? Then continue. The shop is open — now give the vendor full control over what's in it.*

---

Next: with a store open, build complete product management — create, edit, publish — on top of all your guards. → **[Chapter 32 — Product create and edit](32-product-create-edit.md)**
