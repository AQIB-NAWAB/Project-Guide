# Chapter 17 — Errors and status codes

Validation now rejects *malformed* input with a clean, consistent `400` (Chapter 16). But plenty of perfectly well-formed requests still fail — for entirely different reasons. A customer asks for `GET /products/<a-valid-uuid-that-doesn't-exist>`. A vendor tries to edit a product that isn't theirs (Week 2). Someone registers an email that's already taken. These aren't *malformed* requests; they're valid requests with an unhappy answer. Right now your app probably handles them badly — leaking a stack trace as a `500`, or returning some ad-hoc shape a client can't predict.

This chapter gives **every** outcome — success and every kind of failure — the **right HTTP status code** and **one consistent shape**. It's the capstone of Week 1: after it, your API speaks HTTP *correctly*, so any client, proxy, or monitoring tool can understand what happened just from the response. And it's a real step up in difficulty — you're building a small error architecture, not handling one case.

## Where we're headed

By the end you'll have a **centralized error-handling layer**: a small set of meaningful errors your handlers *throw*, **one place** that maps each to the correct status code and the consistent error shape, **safe `500`s** that never leak internals (but are fully logged), and **correct success codes** (`201` when you create a product, not `200`).

## Step 1 — Valid requests fail too — and right now they fail badly

Three tempting ways to handle a failure, each wrong, each teaching what the right way must do.

**Wrong 1 — return `200 OK` with `{ "success": false }`.** The status code *lies*. This is the classic anti-pattern, and it's worse than it looks: the HTTP status is how *machines* understand the outcome — your frontend, caches, proxies, load balancers, and monitoring all read the code first. A `200` that actually failed breaks every one of them; your frontend has to dig into the body to discover the request didn't work, and your monitoring reports a healthy success rate during an outage. Status codes are not decoration — they're the contract.

**Wrong 2 — let the error bubble unhandled.** The framework catches it and returns a generic `500` with a **stack trace** in the body. Two failures at once: the *code* is wrong (a product that doesn't exist is not *your server* breaking — it's the client asking for something absent), and you've just **leaked your internals** to anyone — file paths, library versions, maybe a database string — exactly the leak Chapter 7 warned against for the health endpoint.

**Wrong 3 — `try/catch` in every handler, formatting responses by hand.** It "works," but it's the Chapter 16 rot again: duplicated in every handler, and each one invents its own "not found" shape and maybe its own code. Inconsistency by construction.

The right approach mirrors validation: handlers **throw meaningful errors**, and **one central layer** maps them to the correct code and the consistent shape.

## Step 2 — Status codes: say what actually happened

Before the architecture, get fluent in the vocabulary. HTTP status codes come in families, and the most important idea is *whose fault* each family signals:

| Code | Meaning | In the marketplace |
|---|---|---|
| `200 OK` | Success | A product fetched, an order listed |
| `201 Created` | Success, and a new thing exists | A product was created |
| `204 No Content` | Success, nothing to return | A product deleted |
| `400 Bad Request` | Malformed input | Validation failed (Chapter 16) |
| `401 Unauthorized` | Not authenticated | No/invalid token (Week 2) |
| `403 Forbidden` | Authenticated, but not allowed | A vendor edits another's product (Week 2) |
| `404 Not Found` | The thing isn't there | `GET` a product id that doesn't exist |
| `409 Conflict` | Clashes with current state | Registering an email already taken (Week 2) |
| `429 Too Many Requests` | Rate limited | Too many requests (Week 4) |
| `500 Internal Server Error` | *Your* code broke | An unexpected bug |
| `503 Service Unavailable` | *You* can't serve right now | A dependency down (Chapter 7) |

The line that runs through the whole table: **`4xx` means the client's fault** (an expected outcome — they asked wrong; don't alert on it), **`5xx` means *your* fault** (a genuine bug or outage — log it and wake someone). That distinction is not pedantry; in Step 5 it decides what you log and what pages you. And the headline lesson: **a missing product is `404`, never `500`** — the client asked for something that isn't there, your server worked perfectly.

## Step 3 — One error contract for everything

Extend the validation shape from Chapter 16 so that *every* failure — not just validation — returns the same structure, with the right code:

```
{ "error": { "type": "<machine-readable type>", "message": "<human-readable>", ...optional details } }
```

So across your API:

```
404  { "error": { "type": "not_found",        "message": "Product not found" } }
403  { "error": { "type": "forbidden",        "message": "You don't own this product" } }
409  { "error": { "type": "conflict",         "message": "Email already registered" } }
400  { "error": { "type": "validation_error", "message": "Invalid input", "fields": { ... } } }
500  { "error": { "type": "internal",         "message": "Something went wrong" } }
```

The Chapter 16 validation error wasn't a special case after all — it was the *first instance* of this one contract. Now the client writes a single error-handling path: read the status code for the category, read `error.type` for the specifics. One shape, every endpoint.

## Step 4 — Throw meaning, map centrally

Here's the architecture, and its heart is a clean separation: **handlers describe *what* went wrong in domain terms; one central layer decides *how* that looks in HTTP.**

Define a small set of **domain error classes**, each carrying its status code and type — `NotFoundError` → `404`, `ForbiddenError` → `403`, `ConflictError` → `409` (and the validation error from Chapter 16). A handler that can't find a product simply throws:

```
// in a handler — it expresses the DOMAIN problem, and knows nothing about HTTP
if (!product) throw new NotFoundError('product')
```

Then **one error-handling middleware, registered *last* in the pipeline**, catches everything that's thrown, recognises your domain errors, and renders the right code and the consistent shape:

```
// registered after all routes — the single place HTTP error responses are produced
app.use(errorHandler)   // you write errorHandler; it maps thrown errors → status + shape
```

The payoff is that your handlers stay clean and *domain-focused* — they never build a `404` response, they just say "this product isn't here" — and there's exactly one place that knows how errors become HTTP. Change the error shape once, and every endpoint changes with it.

## Step 5 — The catch-all: safe `500`s that don't leak

What about an error that *isn't* one of your known domain errors — a real bug, a dropped database connection, a `null` you didn't expect? The same central handler is your safety net, and it must do two things at once:

- **To the client:** return a **generic `500`** in the consistent shape, with **no internal detail** — no stack trace, no database message, nothing an attacker could use (the Chapter 7 rule, now enforced for every route at once).
- **To your logs:** record the **full** error — stack trace, message, and the request context (the request id you'll add in Chapter 57) — so *you* can debug it.

The client gets a safe, vague message; you get everything. And this is where the `4xx`/`5xx` distinction earns its keep: `4xx` errors are *expected* (a client asked for a missing product — log at most an info line, never alert), while `5xx` errors are *your bugs* (log loudly, alert, fix). Wire your logging to that split and your alerts mean something.

## Step 6 — Do it on your project (hands-on)

Build the layer and prove each kind of outcome:

**1. Write the central error-handling middleware** and register it **last**, after all routes. It maps known domain errors to their status + the consistent shape, and falls back to a safe `500` for everything else.

**2. Define a few domain error classes** — `NotFoundError`, `ForbiddenError`, `ConflictError` — and fold Chapter 16's validation error into the same contract so it renders identically.

**3. Make `GET /products/:id` return a real `404`.** When the id is a valid UUID but no such product exists, throw `NotFoundError` and let the central handler respond:

```bash
curl -i localhost:3000/products/<valid-but-nonexistent-uuid>
# → 404  { "error": { "type": "not_found", "message": "Product not found" } }   (NOT 500, NOT 200)
```

**4. Return the correct success code.** Make `POST /products` respond **`201 Created`** on success, not `200`. And confirm the validation `400` from Chapter 16 now flows through this *same* contract:

```bash
curl -i -X POST localhost:3000/products -H 'content-type: application/json' -d '{...valid...}'   # → 201
curl -i -X POST localhost:3000/products -H 'content-type: application/json' -d '{"title":""}'     # → 400, same shape
```

**5. Prove the safe `500`.** Temporarily throw an unexpected error inside a handler and hit it: confirm the client gets a **generic `500` with no stack trace**, while your **server logs hold the full error**. Then remove the temporary throw.

**6. Confirm you're ready for next week.** Your contract already defines `unauthorized` (401), `forbidden` (403), and `conflict` (409) — so when auth arrives in Week 2, throwing `new ForbiddenError()` or `new ConflictError()` already produces correct, consistent responses. The error layer is built *ahead* of the features that need it.

> 💡 **Hint — async errors can slip past the handler.** In Node, an error thrown inside an `async` route may not reach your error middleware automatically, depending on your framework — it can surface as a hung request or an unhandled promise rejection instead of your tidy `404`. If a thrown `NotFoundError` doesn't produce a `404`, you likely need to forward async errors to the error handler (an `asyncHandler` wrapper, or your framework's built-in async support).

> **📖 Mandatory read — before Chapter 18.** Read the **MDN HTTP status code reference** (skim the `4xx` and `5xx` lists — know what each common code means) and your framework's **centralized error-handling** docs. *Required: Week 2's auth and authorization lean entirely on returning the right `401`/`403`/`409` through this layer.*

> **Interesting to read.** Whole APIs have been built that return `200 OK` for everything and bury the real outcome in the body — and the downstream pain is legendary: monitoring that can't see failures, retries that never trigger, clients riddled with special-case parsing. Search *"why returning 200 for errors is bad"* (and revisit **RFC 7807** from Chapter 16) to see the discipline you just built treated as a first principle.

## Definition of Done

Things you can **see or run** — and the gate to Week 2:

- [ ] A **centralized error handler** (registered last) produces **one consistent error shape** for every failure
- [ ] Handlers **throw meaningful domain errors** (`NotFoundError`, etc.) and never format HTTP responses themselves
- [ ] `GET /products/:id` for a valid-but-missing id returns **`404`** (you tested it — not `500`, not `200`)
- [ ] `POST /products` returns **`201`** on success; the Chapter 16 validation case returns **`400`** through the *same* contract
- [ ] An unexpected error returns a **generic `500` with no internal details**, while the **full error is logged** server-side
- [ ] You can state the **`4xx` vs `5xx`** distinction and why it governs logging and alerting
- [ ] The contract already supports the **`401`/`403`/`409`** types Week 2 will need

> **✍️ Log it (mandatory).** In `learning-log/17-errors-and-status-codes.md` — **decision** first, then **topics**: **(Decision)** Why route all failures through one central error layer with thrown domain errors, instead of `try/catch` in each handler — and why is returning `200` with `{success:false}` wrong? **(Topics)** (1) Explain the `4xx` vs `5xx` distinction and how it should govern what you log and alert on. (2) Why is a missing product a `404` and not a `500`? (3) Why must a `500` hide its details from the client but log them fully on the server? (4) How does *throwing* a `NotFoundError` (instead of building a response in the handler) separate domain logic from HTTP concerns?

*All boxes ticked and the log written? **That completes Week 1.** You have a configured, containerized service with a self-defending data model, validated inputs, and an API that speaks HTTP correctly — a genuine production foundation. Week 2 builds identity on top of it: who is this user, and what are they allowed to touch?*

---

Next: your API is correct and predictable — but it has no idea *who* is calling it. Week 2 begins with the question every secure system must answer first. → **[Chapter 18 — The authentication threat model](18-auth-threat-model.md)**
