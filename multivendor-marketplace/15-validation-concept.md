# Chapter 15 — Validation: guarding the boundary

Your data module is done, and your tables defend themselves: the constraints from Chapter 11 mean the database will *reject* a negative price or a reference to a store that doesn't exist. But the database is the **last** line of defence — the bouncer at the very back of the building. Nothing yet checks the crowd at the **front door**. Right now, data arriving from the outside world — a vendor submitting a new product, a customer placing an order — flows inward with nobody inspecting it until it slams into a database constraint and produces an ugly crash. This module is about the front door, and this chapter is the principle behind it.

It's a concept chapter — you'll build the real request validation next chapter. Here you learn *why* validation exists, *where* it belongs, and what it is and isn't. And you've actually done it once already: back in Chapter 6 you validated your *config* with a zod schema and refused to boot on bad input. That was validation at a boundary. Now you generalise that exact move to every request your app will ever receive.

## What you'll be able to explain

By the end you can describe the **trust boundary** and why all external input is untrusted; why TypeScript types can't guard it; the difference between validation, authorization, and business rules; how validation defends against mass-assignment attacks; and why app-level validation and database constraints are partners, not duplicates.

## Section 1 — The trust boundary

Draw a line around your application. *Outside* the line is everything you don't control: HTTP request bodies, query strings, route parameters, headers, and later the responses of external APIs and the payloads of queued jobs. All of it is **untrusted** — it can be missing, malformed, the wrong type, malicious, or simply wrong. *Inside* the line is where you want to live in peace, writing code that trusts its data.

**Validation is the gate on that line.** Its job is to take untrusted input and either turn it into *trusted, well-formed data* or reject it at the door with a clear error. The promise this gives you is enormous: every piece of code *past* the gate can assume the data is clean. No defensive checks scattered through your services, no "what if price is a string here" — because nothing malformed ever got in.

```
   UNTRUSTED                  │  gate  │              TRUSTED
   request body, query,       │ ┌────┐ │   clean, typed, well-formed data
   params, headers,    ──────▶│ │ ✓? │ │──▶  your services & database can
   external responses         │ └────┘ │   simply trust it
                              the validation boundary
```

## Section 2 — Why you can't skip the gate

Three tempting shortcuts, each of which fails — and seeing *how* tells you what the gate must do.

**Shortcut 1 — just trust the input.** A request arrives to create a product with `price_cents: "-5"`, `quantity: 0`, `title: {}`, and a bonus field `role: "admin"`. Trust it and that garbage flows toward your database. *Some* of it the constraints will catch (the negative price), but as an ugly `500` deep in a handler — and some of it they won't (a structurally-valid but nonsensical email, or that extra `role` field). Trusting input means crashes, corrupt data, and security holes.

**Shortcut 2 — rely on TypeScript types.** You wrote `req.body` as `{ price: number }`, so surely it's a number? **No** — and this is the trap from Chapter 3 returning. TypeScript types are **erased at runtime**; they're a compile-time fiction that vanishes when the code actually runs. At runtime, `req.body` is `unknown` — `price` can be a string, `undefined`, or an object, and the `: number` annotation was a promise the compiler believed but cannot enforce. The boundary is crossed at *runtime*, exactly where types no longer exist. This is *why* you needed zod for config: a type is not a runtime check; a schema is.

**Shortcut 3 — scatter `if`-checks in each handler.** Hand-write `if (!req.body.email) return error` in every route. It "works" but rots: it's inconsistent between endpoints, duplicated everywhere, forgotten on the one handler that matters, and there's no single declarative answer to "what *is* a valid create-product request?"

## Section 3 — The right approach: a schema at the boundary

The professional move is one **declarative schema per entry point** that describes exactly what valid input looks like — required fields, types, formats, ranges, allowed values — and rejects everything else with a clear, structured error. It's the config pattern from Chapter 6, pointed at requests instead of the environment: one place, declarative, enforced at runtime.

But there's a deeper idea in *how* a good schema works, and it has a name worth knowing.

## Section 4 — Parse, don't validate

The principle is called **"parse, don't validate,"** and the distinction is sharp. A weak gate *checks* the input (yes/no) and then hands the **original raw input** onward — so downstream code still receives `{ price: "1999" }`, a string, and has to remember to convert it. A strong gate **parses**: it transforms the input into a *new, trusted, typed value* — `{ price: 1999 }`, an actual number — coercing types, trimming strings, applying defaults, and dropping anything not in the schema. The raw, untrusted shape *never travels inward at all*; only the parsed, clean object does.

You already saw this in Chapter 6: `z.coerce.number()` turned the string `"3000"` into the number `3000` before any code used it. That's parsing. The output of validation isn't a thumbs-up — it's a better object than the one that came in.

## Section 5 — Validation is not authorization, and not business rules

People cram three different gates into one and make a mess. Keep them distinct, and in this order:

1. **Validation** — *is this input well-formed?* Structure, type, format, range, allowed values. Cheap, synchronous, needs no database. ("`price_cents` is a non-negative integer; `email` looks like an email; `title` is a non-empty string.")
2. **Authorization** — *is this user allowed to do this?* (All of Week 2.) A perfectly well-formed request can still be forbidden.
3. **Business rules** — *does this make sense right now?* Often needs the database and context. ("Is this product in stock? Is the vendor verified?")

Validation is the first and cheapest gate. Don't smuggle "is the vendor verified" into your input schema — that's a business rule that needs the database, and mixing it in makes both jobs muddier. The input schema answers one question only: *is this shaped like a valid request?*

## Section 6 — Reject the unexpected: mass assignment

Here's the security beat, and it's a real attack. Suppose your create-product handler takes `req.body` and spreads it straight into a database insert. A user sends the fields you expected — plus a few you didn't: `isVerified: true`, or `role: "admin"`, or `vendor_id: "<someone-else's>"`. If your code blindly trusts the *shape* of the input, those extra fields ride along and set things the user was never meant to control. That's **mass assignment**, and it has caused real breaches.

Validation is the defence, because a schema is an **allowlist**: only the fields it declares are permitted through, and unknown fields are stripped or rejected. The user can send `role: "admin"` all day; it never makes it past the gate, because the gate only knows about the fields *you* said were valid. Never trust input to contain only what you intended — make the schema decide what's allowed in.

## Section 7 — Defence in depth: the gate and the bouncer

So if the database already rejects bad data (Chapter 11), why validate at the boundary at all? Because they do *different jobs*, and you want both:

- **App validation (the front-door gate)** is *expressive, friendly, and early*. It catches malformed input the database shouldn't even care about (an email that isn't an email), returns a clean, helpful `400` describing *what* was wrong, and stops bad data long before it reaches the database.
- **Database constraints (the back-door bouncer)** are the *guarantee* — the integrity that holds even if validation is bypassed by another service, a script, or a bug. They don't give nice errors; they make corruption *impossible*.

This is the Chapter 11 principle stated whole: **validate in the app for a good experience, enforce in the database for an absolute guarantee.** A well-formed-but-wrong request should be rejected at the boundary with a tidy error message — never allowed to limp inward and explode against a constraint as a `500`.

## Section 8 — Map your trust boundary (hands-on)

Make this concrete for *your* app. List every place untrusted data currently enters, or soon will:

- **HTTP request bodies** — creating a product, registering, placing an order.
- **Query parameters** — search terms, filters, pagination cursors (Week 3).
- **Route parameters** — the `:id` in `/products/:id` (untrusted: is it even a UUID?).
- **Headers** — later, the auth token (Week 2).
- **Later:** external API responses, webhook payloads, and the job payloads your worker will consume (Week 4) — *these are boundaries too*, and people forget them.

For each, write down one thing: *is anything validating it yet?* Right now the honest answer is "only my config." That list is your map of where Chapter 16 will install gates — and a reminder that the boundary is wider than just request bodies.

## Section 9 — Build it on your project: guard your first endpoint (hands-on)

Principle is nothing without a guarded endpoint to show for it, so make it real on *one* route now. You've had a web framework wired up since the `/health` route in Chapter 7, a `products` table since Chapter 12, and seed data since Chapter 13 — everything you need to create your first **write** endpoint and watch the boundary do its job.

**1. Create the endpoint — unguarded, on purpose.** Add a route to create a product: **`POST /products`**, taking the product's details in the JSON body (`title`, `price_cents`, `currency`, `status`, and the JSONB `attributes`). *Which* store it belongs to will come from the authenticated vendor in Week 2 — for now, associate it with one of your seeded stores and leave a `// TODO: derive store from auth` note. Don't validate anything yet; just have it insert what it's given.

**2. Feel the problem.** Send it deliberately broken requests and watch it fail badly:

```bash
# a negative price, an empty title, an invalid status, and a sneaky extra field
curl -X POST localhost:3000/products -H 'content-type: application/json' -d '{
  "title": "",
  "price_cents": -500,
  "status": "on_sale",
  "role": "admin"
}'
```

One of two bad things happens: it crashes deep in your handler, or it reaches the database and bounces off a Chapter 11 constraint as an ugly `500`. `status: "on_sale"` isn't even a real status, and `role: "admin"` has no business being there — yet nothing stopped any of it at the door.

**3. Install the gate.** Add a validation schema for this endpoint's body — the same zod move you made for config in Chapter 6, now pointed at the request. The schema *is* your contract for "a valid create-product request":

```
// the SHAPE of the contract — you write the schema; this is the spec, not the handler
CreateProductBody:
  title        → non-empty string, trimmed
  price_cents  → integer, >= 0
  currency     → 3-letter code (e.g. 'USD')
  status       → one of 'draft' | 'published'        // 'on_sale' is rejected
  attributes   → object (the JSONB tail), optional, defaults to {}
  // unknown keys (like `role`) are STRIPPED — the schema is an allowlist
```

Parse the body against it at the very top of the handler: on failure, return **`400`** with the list of what's wrong; on success, hand the **parsed, typed** object to your insert — never the raw `req.body`.

**4. Prove it works.** Re-send the broken request from step 2 — you now get a clean `400` naming the problems, not a `500`. Send a *valid* request and the product is created. Send one carrying `role: "admin"` or `isVerified: true` and confirm those fields are silently dropped — the mass-assignment door is shut. You've turned an abstract principle into a guarded endpoint you can demo.

**5. See what you haven't covered yet.** This is *one* endpoint. Look back at your Section 8 boundary map: every other entry point — other request bodies, the `:id` in a route, query params, later job payloads — is still wide open. Turning this one-off into a reusable pattern across all of them, validating params and queries too, and returning a *consistent* error shape, is exactly the next chapter's job.

> **📖 Mandatory read — before Chapter 16.** Read the short, canonical essay **"Parse, don't validate"** (search that exact phrase — by Alexis King), and skim your validation library's docs on **parsing and safe-parsing** input. *Required: the next chapter builds request validation directly on this principle.*

> **Interesting to read.** Mass assignment isn't theoretical — in 2012 a developer demonstrated the flaw on **GitHub itself** by adding his own key to a project he didn't own, simply by submitting an extra field the server trusted. Search *"GitHub mass assignment 2012"*. It's the clearest possible argument for treating every incoming field as something you must explicitly allow, never implicitly trust.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 16:

- [ ] A **`POST /products`** route exists and creates a product in your seeded marketplace
- [ ] Its request body is **validated by a schema at the boundary**; you tested a negative price, an empty title, and an invalid status and got a clean **`400`** (not a `500`)
- [ ] The handler works with the **parsed, typed** value — it never reads raw `req.body`
- [ ] **Mass assignment is blocked**: you sent `role: "admin"` (or `isVerified: true`) and confirmed the field was stripped, never persisted
- [ ] A **valid** request successfully creates the product
- [ ] You can point at **every other entry point still unguarded** on your boundary map — the work Chapter 16 generalises

> **✍️ Log it (mandatory).** In `learning-log/15-validation-concept.md` — **decision** first, then **topics**: **(Decision)** Why does *this* marketplace need to validate input at the boundary when the database already rejects bad data — what does each gate do that the other can't? Why can't you rely on your TypeScript types to do it? **(Topics)** (1) What is the trust boundary, and what can code *past* it safely assume? (2) Explain "parse, don't validate" — using your `POST /products` schema, what's different about the object that left the gate versus the one that arrived? (3) What is a mass-assignment attack, and how did your allowlist schema stop the `role` field? (4) Give one example each of a *validation*, an *authorization*, and a *business-rule* concern for this product-creation endpoint.

*All boxes ticked and the log written? Then continue. You've guarded one endpoint by hand — next you turn it into a pattern every route shares, with a consistent error shape.*

---

Next: take the principle and make it run — one schema per endpoint, rejecting bad requests with clean errors before they reach your handlers. → **[Chapter 16 — Request validation](16-request-validation.md)**
