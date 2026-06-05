# Chapter 27 — Multi-tenant isolation: the concept

Roles now stop the wrong *kind* of user (Chapter 26) — but you ended that chapter on an uncomfortable truth: a vendor who passes `requireRole('vendor')` can still reach *another vendor's* product by putting its id in the URL. Role checks guard the *action*; nothing yet guards the *specific object*. In a single-tenant app that gap barely matters. In a **marketplace**, where hundreds of independent makers share one database, it's the most important security property you'll build — because a single missed check leaks one vendor's private data, orders, or payouts to another.

This is **multi-tenant isolation**: many tenants (vendors) share one app, but each is sealed off so completely that no vendor can see or touch another's data. It's the marketplace form of the object-level authorization problem from Chapter 25, and it's the heart of this module (Chapters 27–30). This concept chapter gives you the model; the next three build and *prove* it.

## What you'll be able to explain

By the end you can define multi-tenancy and the tenant boundary for this app; compare the ways to isolate tenants; explain the two layers of defence (query scoping and ownership checks); and describe why a single forgotten scope is a breach.

## Section 1 — What "tenant" means here

A **tenant** is one isolated customer of a shared system. In your marketplace, **each vendor is a tenant**, and the **store is the tenant boundary** — exactly the ownership line you modelled back in Chapter 8 (`store.vendor_id`) and extended in Chapter 9 (`product.store_id`). Every vendor-owned record traces back through that chain to one vendor: a product belongs to a store, a store belongs to a vendor, an order belongs to a store. "Is this *yours*?" is always answered by following that chain to `req.user`.

Isolation is the guarantee that **every operation on vendor-owned data is confined to the current tenant.** Vendor A's queries return only A's rows; A's writes touch only A's rows; A asking for B's product gets *nothing*.

## Section 2 — How to isolate tenants (the approaches)

There's a real architectural choice in *how* tenants share infrastructure:

| Approach | Isolation | Cost | Verdict |
|---|---|---|---|
| **Database per tenant** | Strongest (physical) | A new DB per vendor — unmanageable at scale | No |
| **Schema per tenant** | Strong | Hundreds of schemas, migration pain | No |
| **Shared database, row-level scoping** | Logical — every row carries a tenant key, every query filters by it | One schema, one migration set; isolation enforced in code (+ optionally the DB) | **Yes** |

**The course uses the shared-database, row-scoped approach** — the standard for SaaS marketplaces. All vendors share your tables; isolation comes from *every query being scoped* to the current tenant via the ownership chain. It scales to thousands of vendors with one schema — but it puts the responsibility on *you* to never forget the scope, which is exactly why this module is so careful.

## Section 3 — Two layers of defence

Isolation isn't one check; it's two complementary layers, and you'll build one in each of the next chapters:

- **Layer 1 — Query scoping (Chapter 28).** Every *read and write* of vendor-owned data carries the tenant filter by default: `WHERE store_id = <my store>`. Make scoping the default path, not something you remember to add, so a non-owned row is simply *invisible* — it doesn't come back from the query at all.
- **Layer 2 — Ownership checks on object access (Chapter 29).** For operations that target a specific object by id (edit, delete), explicitly confirm the object belongs to the current tenant before acting — closing the **IDOR** hole from Chapter 26.

These reinforce each other. Scoping means most code *can't* see other tenants' rows; ownership checks catch the per-object operations where an id from the client must be validated against the tenant. Two layers, so one slip isn't fatal — defence in depth (Chapters 11, 15, 18) applied to data.

## Section 4 — The failure mode: the confused deputy

The attack you're defending against has a name: the **confused deputy**. Your server is *authenticated as vendor A* and is trusted — but if it acts on vendor B's object because it never checked ownership, A has used the server as a "deputy" to reach data they could never access directly. **IDOR** (insecure direct object reference) is the concrete form: change `/products/<my-id>` to `/products/<their-id>` and the server, checking only that you're *a* vendor, happily edits someone else's product.

Two truths follow. First, **authentication and roles don't prevent this** — A is genuinely a logged-in vendor; the missing check is *ownership*. Second, **opaque UUIDs (Chapter 8) don't prevent it either** — they make ids hard to *guess*, but a leaked or shared id still works if nothing enforces ownership. Hard-to-guess is not the same as access-controlled. Isolation is the enforcement.

## Section 5 — A backstop worth knowing: row-level security

App-level scoping is your primary defence, but it has a weakness: it relies on *you* writing the scope on every query, forever. A defence-in-depth backstop exists — **Postgres Row-Level Security (RLS)** — where the *database itself* refuses to return rows that don't match the current tenant, so even a query that *forgot* its scope returns nothing. It's more advanced to operate (you set a tenant context per connection) and optional for this course, but worth knowing as the belt-and-braces option real multi-tenant systems reach for. (Mention it in your notes; you may add it as a bar-raiser.)

## Section 6 — Map isolation for your project (hands-on)

Before building, write down exactly what must be isolated.

**1. Create `docs/ISOLATION.md`.** List every **vendor-owned resource** and its **scope key** (the column that ties it to a tenant):

```
Resource     | Scope key                     | Operations to scope
-------------|-------------------------------|----------------------------------
products     | store_id → my store           | list, get, edit, delete
orders       | store_id → my store           | list, get, update status
store        | vendor_id → me                | get, edit
order_items  | via order.store_id            | read (through its order)
```

**2. Audit current exposure.** Walk your existing endpoints and mark which already scope and which don't. Today: `POST /products` derives the store from `req.user` (good — Chapter 24), but `GET /products/:id` and any future edit/delete do **not** check ownership — an open IDOR hole. Write down each gap; these are precisely what Chapters 28–29 close, and Chapter 30 will *test*.

**3. Tick the threat model.** In `THREAT-MODEL.md` (Chapter 18), the "acting beyond your role / IDOR" row is this module — link it to `ISOLATION.md`.

Commit both. You now have the spec for the next three chapters.

> **📖 Mandatory read — before Chapter 28.** Read the **OWASP "Broken Object Level Authorization"** entry (the #1 API risk) and a short intro to **multi-tenant data isolation** (search *"multi-tenant SaaS data isolation patterns"*). *Required: Chapter 28 implements row-level scoping, the primary defence described here.*

> **Interesting to read.** The "confused deputy" is a classic security concept from the 1980s that perfectly describes modern IDOR breaches — and it's why a trusted, authenticated server is *exactly* the thing attackers try to trick into fetching data on their behalf. Search *"confused deputy problem"* to see the decades-old idea behind today's #1 API vulnerability.

## Key takeaways

You can now explain:

- **Multi-tenant isolation**: many vendors share one database, but each is sealed off, with the **store as the tenant boundary** (the Chapter 8 ownership chain).
- The **shared-DB, row-scoped** model is the practical choice, and it makes *you* responsible for scoping every query.
- Isolation has **two layers** — query scoping (default tenant filter) and ownership checks (per-object) — that reinforce each other.
- **Authentication, roles, and opaque UUIDs don't prevent IDOR** — only an enforced ownership/scope check does; the attack is the *confused deputy*.

> **✍️ Log it (mandatory) — this is the gate.** In `learning-log/27-isolation-concept.md` — **decision** first, then **topics**: **(Decision)** Why does this marketplace use a shared database with row scoping instead of a database per vendor, and what responsibility does that put on you? For your app, what is the "tenant boundary," and which column expresses it for products and for orders? **(Topics)** (1) Explain the two layers of isolation and why both are needed. (2) Why don't authentication and role checks prevent IDOR? (3) Why don't opaque UUIDs (Chapter 8) make ownership checks unnecessary? (4) What is the "confused deputy," and how does it describe an IDOR attack on your API?

*Answered all of it, and committed `ISOLATION.md`? Then continue. You know what must be sealed off — now make scoping the default for every query, so other tenants' data is simply invisible.*

---

Next: build the first isolation layer — scope every query to the current vendor so another tenant's rows never even come back. → **[Chapter 28 — Query scoping](28-query-scoping.md)**
