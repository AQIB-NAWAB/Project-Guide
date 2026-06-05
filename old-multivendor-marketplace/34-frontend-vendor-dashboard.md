# Chapter 34 — The vendor dashboard (frontend)

Every vendor feature now exists as an API — onboarding, products, listing, all secured and isolated. But a maker isn't going to use `curl`. This chapter puts a face on the backend: a **vendor dashboard** that logs in, shows the vendor their products, and lets them create, edit, and publish — by *consuming* the API you built. It's a frontend/layer chapter (the guide explicitly supports these), and its real lessons aren't about CSS — they're the integration realities every client of a secure API must handle: the **token lifecycle**, where to keep credentials, and how to turn your structured errors into a good user experience.

You can build this in whatever frontend you're comfortable with — a small SPA, a server-rendered app, even plain HTML and `fetch`. The course doesn't mandate a frontend framework; it mandates that the dashboard handles the auth and error contracts *correctly*. That's the teaching, and it's where most real frontends get auth subtly wrong.

## Where we're headed

By the end you have a working vendor dashboard that: logs in and manages the access/refresh token lifecycle (Chapters 21–22); routes a storeless vendor to onboarding (Chapter 31); lists the vendor's products and supports create/edit/publish; and renders your `400`/`401`/`403` responses as sensible UI.

## Step 1 — The screens, by what they *do*

Keep the UI description at the level of behaviour, not buttons:

- **Login** — email + password → calls `POST /auth/login`, receives the access token, lands the vendor in the dashboard.
- **Onboarding** — if the vendor has no store yet (Chapter 31), route them here to open one before anything else.
- **Product list** — the vendor's own products (Chapter 33), filterable by status; the shop at a glance.
- **Create / edit product** — a form posting to `POST /products` / `PATCH /products/:id`, including the JSONB `attributes` for the product type.
- **Publish toggle** — flips `status` via `PATCH` (Chapter 32).

Each screen is a thin shell over an endpoint you already built and secured.

## Step 2 — The token lifecycle (the real work)

This is where frontends go wrong, so handle it deliberately, following your Chapter 21–22 decisions:

- **Store the access token in memory** (a variable/state), not `localStorage` — minimizing XSS exposure (Chapter 21). Attach it as `Authorization: Bearer <token>` on every API call.
- **The refresh token lives in the `httpOnly` cookie** (Chapter 22) — your JavaScript never touches it; the browser sends it automatically to `/auth/refresh`.
- **On a `401`, refresh and retry.** When an API call returns `401` (the access token expired), the client calls `POST /auth/refresh` to get a new access token, then *retries the original request once*. If the refresh itself fails (`401`), the session is truly over — send the user to login. This silent refresh is what makes a 15-minute access token invisible to the user.

```
api call → 401 ?
   → POST /auth/refresh
       → 200: new access token → retry the original call once
       → 401: refresh expired/revoked → redirect to login
```

Getting this loop right is the single most important thing in the chapter — it's the client half of the two-token system you built server-side.

## Step 3 — Turn structured errors into UX

Your Chapter 16–17 error contract was designed for *exactly* this moment — one consistent shape the client can rely on. Map each to UI:

- **`400` validation_error** — the `fields` map (Chapter 16) lets you show each message *next to its field* ("price must be ≥ 0" under the price input). This is why you built a field-keyed shape.
- **`401`** — trigger the refresh-and-retry loop (Step 2); only show "please log in" if refresh fails.
- **`403`** — "you're not allowed to do this" (e.g. an unverified vendor trying to open a store → prompt them to verify; Chapter 23).
- **`409`** — a specific conflict message ("you already have a store", "that slug is taken").
- **`404`** — "not found" (and recall, for someone else's product this is *intentional* — Chapter 29).

Because every endpoint returns the same shape, you write this mapping **once** and reuse it — the payoff of the consistent contract.

## Step 4 — Isolation is visible in the UI

A quiet but reassuring property: because the API is tenant-scoped (Chapters 28–30), the dashboard *automatically* only ever shows the logged-in vendor's products. You don't filter by vendor on the client — you *can't* accidentally show another vendor's data, because the server never sends it. The frontend trusts the backend's isolation; security lives on the server, never in the UI. (A good thing to verify: log in as two different vendors and confirm each dashboard shows only their own shop.)

## Step 5 — Do it on your project (hands-on)

**1. Build the login screen** → `POST /auth/login`, keep the access token in memory, store nothing sensitive in `localStorage`.

**2. Implement the refresh-and-retry interceptor** (Step 2): wrap your API calls so a `401` transparently refreshes and retries once, and a failed refresh sends the user to login.

**3. Route the storeless vendor to onboarding** (Chapter 31) before showing the product UI.

**4. Build the product list + create/edit forms**, posting to your endpoints, including the JSONB attributes. Render `400` field errors inline.

**5. Verify the whole flow and isolation:**

```
log in → (open store if needed) → see only YOUR products → create a draft →
edit attributes → publish → log in as a different vendor → see a completely different shop
```

Let the access token expire (or shorten it) and confirm a request *silently* refreshes and succeeds rather than logging the user out — that's the two-token system felt from the user's side.

> 💡 **Hint — the frontend is not a security boundary.** Never rely on hiding a button to enforce permissions — a determined user calls the API directly. The dashboard hides the "delete" button on others' products for *UX*, but the *security* is the server's ownership check (Chapter 29). Treat the UI as convenience over an already-secure API; every real rule is enforced server-side, and your Chapter 30 test is what guarantees it.

> **📖 Mandatory read — before Chapter 35.** Read about **handling JWT refresh on the client** (search *"axios interceptor refresh token retry"* or your HTTP client's equivalent). *Required: the refresh-and-retry loop is the most error-prone part of consuming a token-based API.*

> **Interesting to read.** A common frontend security mistake is enforcing access control in the UI alone — hiding admin features client-side while the API leaves them open. Search *"client side security is not security"* to see why your dashboard must assume the server does all real enforcement.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 35:

- [ ] A vendor can **log in** and reach a dashboard; the access token is kept **in memory**, the refresh token in the `httpOnly` cookie
- [ ] A **`401` silently triggers refresh-and-retry**; only a failed refresh sends the user to login (you saw a near-expired token refresh transparently)
- [ ] A **storeless vendor is routed to onboarding** before product management
- [ ] The vendor can **list, create, edit, and publish** their products; `400` validation errors render **per-field**
- [ ] Logging in as two vendors shows **two disjoint shops** — isolation visible, enforced by the server
- [ ] No security depends on the UI — every rule is enforced by the API

> **✍️ Log it (mandatory).** In `learning-log/34-frontend-vendor-dashboard.md` — **decision** first, then **topics**: **(Decision)** Why keep the access token in memory and the refresh token in an `httpOnly` cookie rather than both in `localStorage`? Why is the frontend never a security boundary? **(Topics)** (1) Describe the refresh-and-retry loop and what makes a short-lived access token invisible to the user. (2) How does the consistent error contract (Chapters 16–17) let you write error handling once? (3) Why does the dashboard automatically show only the vendor's own data without client-side filtering? (4) What's the difference between hiding a button for UX and enforcing a permission?

*All boxes ticked and the log written? **That completes the vendor experience and Week 2.** Vendors can securely run their shops. Week 3 turns to the other side — customers browsing a catalogue — and the performance challenges that come with real scale.*

---

Next: vendors are done — now build the customer-facing catalogue across *all* stores, and meet the performance problems that come with it. → **[Chapter 35 — The public catalogue](35-public-catalogue.md)**
