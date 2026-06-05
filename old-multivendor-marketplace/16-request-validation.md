# Chapter 16 — Request validation

Last chapter you guarded one endpoint by hand: `POST /products` parses a schema right inside the handler and returns a `400` on bad input. It works — but now imagine doing that for *every* route in the marketplace. You'd copy-paste the same "parse the body, check if it failed, format an error, pull out the clean data" dance into dozens of handlers. Some would format the error one way, others another. One handler would quietly forget to validate at all. Your clients would face a different error shape on every endpoint. The one-off was the right way to *learn* the gate; it's the wrong way to *run* a hundred of them.

This chapter turns validation from a thing you do in each handler into a **reusable pattern applied to every route** — guarding the body, the route params, *and* the query, and returning one consistent error shape everywhere. It's a step up in difficulty: you're building a small piece of architecture, not just one more schema.

## Where we're headed

By the end you'll have a single reusable **validation middleware** that guards any route from a schema; validation covering **request bodies, route params, and query strings**; and **one consistent `400` error shape** that every endpoint returns on bad input — so your frontend can rely on it.

## Step 1 — Why the one-off doesn't scale

Picture the Chapter 15 approach repeated across the app. Every handler begins with the same ritual: take the raw input, run it through a schema, branch on success or failure, build a `400` body, and only then get to the actual work. Three problems fall out of that, all the rot you met in Chapter 15's "scattered if-checks" — just wearing schemas now:

- **Duplication.** The same parse-and-respond logic is pasted into every handler. Change how you format errors and you're editing fifty files.
- **Inconsistency.** Hand-rolled per handler, the error shape *drifts* — `{ error: "bad" }` here, `{ message: "...", field: "..." }` there. A client can't write one piece of code to read them all.
- **Omission.** Validation that lives *inside* each handler is validation each handler can *forget*. The one route where someone skipped it is the security hole.

The fix is to lift validation *out* of the handlers and into one place that's applied to a route declaratively — so guarding a route is a single, visible, impossible-to-forget step, and the handler only ever runs on clean data.

## Step 2 — The reusable gate: a validation middleware

The tool for "run something before the handler" is **middleware** — a function the framework runs as the request flows toward your handler (the same pipeline your `/health` route already lives in). You'll write one `validate(schema)` middleware that does the whole gate, once:

- It parses the relevant part of the request against the schema you give it.
- On **failure**, it short-circuits — the handler never runs — and responds with your standard `400` shape (Step 4).
- On **success**, it replaces the raw input with the **parsed, typed** value and calls the handler.

Now guarding a route is one declarative line — the wiring (which is a contract, not feature logic):

```
// wiring a route through the gate — you write the middleware and handler bodies
router.post('/products',  validate({ body: CreateProductBody }),  createProductHandler)
router.get('/products/:id', validate({ params: ProductIdParams }), getProductHandler)
```

The handler that follows is blissfully simple: by the time it runs, the input is guaranteed valid and typed. Every route is guarded the same way, and you can *see* the guard sitting on each route at a glance.

## Step 3 — Validate every part of the request, not just the body

Here's where people leave holes. The body is *one* untrusted input — but your Chapter 15 boundary map listed others, and each is just as untrusted:

- **Route params.** In `GET /products/:id`, that `:id` arrives as a raw string. Feed an un-validated `:id` straight into a database lookup and a value like `not-a-uuid` causes a UUID-cast error deep in the query — an ugly `500` for what should be a clean `400`. So validate it: `:id` must be a UUID.
- **Query strings.** `GET /products?status=published&limit=20` — filters and pagination. `limit` must be a positive integer within a sane maximum (or a client asks for `limit=1000000` and strains your database); `status` must be one of the allowed values. Query input is user input.
- **The body.** As before.

Give each part its **own schema** — a params schema, a query schema, a body schema — and let the middleware validate whichever parts a route declares. A route that takes an id *and* a body validates both.

## Step 4 — One consistent error shape

When validation fails *anywhere* — any route, any part of the request — it must return the **same** structured error. Decide that shape now and use it everywhere:

```
400 Bad Request
{
  "error": {
    "type": "validation_error",
    "fields": {
      "title":       ["Required"],
      "price_cents": ["Must be greater than or equal to 0"]
    }
  }
}
```

A field-to-messages map is the most useful form, because your frontend (coming in Week 2) can show each error *next to the field that caused it*. The point is **consistency**: one shape, every endpoint, so a client writes one error-handling path instead of guessing per route. This is the first slice of a broader error contract you'll finish in Chapter 17, which extends the same discipline to *every* kind of error (not found, unauthorized, conflict) with the right status codes. For now, nail the validation case.

## Step 5 — Reuse and compose your schemas

A nice consequence of pulling schemas out of handlers: you can **share** them. Some appear again and again:

- A **UUID-param schema** used by *every* `/:id` route — `/products/:id`, later `/orders/:id`, `/stores/:id`.
- A **pagination-query schema** (`limit`, `cursor`) reused by every list endpoint in Week 3.

Define those once and reuse them, and **compose** larger schemas from smaller shared rules (a shared price rule that enforces "integer ≥ 0", a shared slug rule). This keeps validation both consistent *and* DRY — and as a bonus, since your schemas already describe the exact shape of valid data, they double as the source of truth for your input *types*, so you're not declaring the same shape twice. Reach for a shared schema the second time you'd otherwise copy one.

## Step 6 — Do it on your project (hands-on)

Build the pattern and prove it across more than one route:

**1. Write the `validate` middleware** — one function that takes per-part schemas, parses the matching request parts, responds with the **Step 4 error shape** on failure, and on success attaches the parsed values for the handler.

**2. Refactor `POST /products`** (from Chapter 15) to use it: move the inline body schema out to a named `CreateProductBody`, wire the route through `validate`, and delete the in-handler parsing. Re-send a bad body and confirm you now get the **consistent** error shape:

```bash
curl -X POST localhost:3000/products -H 'content-type: application/json' -d '{"title":"","price_cents":-5}'
# → 400 { "error": { "type":"validation_error", "fields": { "title":[...], "price_cents":[...] } } }
```

**3. Add `GET /products/:id` and guard the param.** Create a minimal read route (the full catalogue is Week 3 — this is just enough to validate a param), wire it through `validate` with a UUID params schema, and test the boundary:

```bash
curl localhost:3000/products/not-a-uuid     # → 400 validation_error (NOT a 500)
curl localhost:3000/products/<a-real-seeded-id>   # → the product
```

That a bad `:id` now returns a clean `400` instead of a database cast error is the whole point of validating params.

**4. Extract a shared schema.** Pull the UUID-param rule into a reusable `IdParams` schema and use it on `GET /products/:id`. You'll reuse it on every `/:id` route from here on.

**5. (If you add a filtered list)** validate a query param — e.g. `?status=` limited to your enum — through the same middleware, proving query inputs go through the identical gate.

**6. Confirm consistency.** Hit every guarded route with bad input — bad body, bad param, bad query — and verify they all return the **same** `400` shape; hit them with good input and confirm they work. Return to your Chapter 15 boundary map: the entries you've now guarded all behave identically.

> 💡 **Hint — middleware order is a classic trap.** Your `validate` middleware reads the parsed body, so it must run *after* your framework's JSON body parser and *before* your handler. If your validator sees `undefined` where a body should be, the body parser isn't running first — check the order in which middleware is registered. (This is the same "is the check running before the handler?" question you'll meet again with auth middleware in Week 2.)

> **📖 Mandatory read — before Chapter 17.** Read your framework's docs on **middleware** (how a function runs in the request pipeline and short-circuits a response) and your validation library's guidance on **validating request parts**. *Required: Chapter 17 builds the full error-handling layer on top of this middleware pattern.*

> **Interesting to read.** There are real *standards* for API error shapes — **RFC 7807 "Problem Details for HTTP APIs"** and the JSON:API error format — precisely because inconsistent error responses are such a common, costly mess. Search *"RFC 7807 problem details"* to see the considered version of the shape you just designed; it's a preview of the broader error contract in Chapter 17.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 17:

- [ ] A **single reusable validation middleware** guards routes from a schema; handlers no longer parse input themselves
- [ ] `POST /products` is **refactored** to use it and still rejects bad bodies — now with the consistent error shape
- [ ] **Route params are validated**: `GET /products/:id` with a non-UUID returns a clean **`400`**, not a `500`
- [ ] Query params are validated **wherever a route has them**
- [ ] **Every** validation failure, on any route or request part, returns the **same structured `400` shape**
- [ ] At least one **shared schema** (the UUID param) is reused across routes
- [ ] Handlers receive only **parsed, typed** data — never raw `req.body`, `req.params`, or `req.query`

> **✍️ Log it (mandatory).** In `learning-log/16-request-validation.md` — **decision** first, then **topics**: **(Decision)** Why move validation into one reusable middleware instead of parsing in each handler — what three problems did the per-handler approach cause? Why must every endpoint return the *same* error shape? **(Topics)** (1) What is middleware, and where in the request pipeline must `validate` run relative to the body parser and the handler? (2) Why do route params and query strings need validation too — give the exact failure an un-validated `:id` causes? (3) Why does a consistent error shape matter to the client that consumes your API? (4) How do shared, composed schemas keep validation DRY, and what second purpose do those schemas serve?

*All boxes ticked and the log written? Then continue. Validation now rejects malformed input consistently — but plenty of *valid* requests still fail for other reasons (a product that doesn't exist, an action you're not allowed to take). Next you give those the same consistent treatment, with the right status codes.*

---

Next: validation handles *malformed* input — but what about a valid request for a product that isn't there? Time for one error contract and the correct status codes across the whole API. → **[Chapter 17 — Errors and status codes](17-errors-and-status-codes.md)**
