# The Multi-Vendor Marketplace Project

> You're building a real, deployed **multi-vendor marketplace** — a place where independent sellers run their own storefronts and fulfil their own orders, while customers browse every seller's catalogue, fill a single cart that spans many sellers, and check out once. It runs on a real URL, over HTTPS, with background workers humming behind it doing the slow work out of sight.

Your build is a marketplace for **independent makers of handmade goods** — small ceramic studios, leatherworkers, print artists — each with their own shopfront, in a clean, calm, editorial style that lets the products breathe. That domain shapes real decisions: products are photo-heavy with wildly varying attributes (a mug vs a satchel), every maker is a separate tenant whose data no one else may touch, and an order spanning three makers has to split into three. **Name the marketplace yourself** — that's your first decision.

## Introduction — what you'll build and why it matters

Almost any developer can build CRUD. Far fewer can answer the questions that actually decide whether software survives production: *What happens to this endpoint under load? How do you stop vendor A from reading vendor B's orders? Where does the order-confirmation email get sent — and what happens when it fails?* Those questions **are** the job, and a marketplace puts every one of them on the table at once.

This course is not about CRUD — you already have that. It's about everything that turns a CRUD app into a **production system**: configuration and secrets, real authentication and multi-tenant isolation, performance under load, moving slow work into background jobs, and shipping it so it stays up.

**When you're done**, you'll have a deployed marketplace where makers manage their own stores, customers shop across makers in one cart, orders split per vendor with the price snapshotted at purchase, and the slow work — emails, image processing, payout reports — runs in background workers, behind caching, rate limiting, and logging you wouldn't be embarrassed to show in an interview.

## How to use this course

Read this part before you start anything — it's how the course works.

### It's a sequence of chapters

The course is **~60 short chapters, grouped into 15 modules across 4 weeks**. Each chapter is one focused topic — a decision to make, a thing to set up, or a feature to build — small enough to finish in a sitting. Work through them **in order**; the full list is in the [course outline](#course-outline) below.

### Each chapter teaches, then asks you to build

A chapter isn't a spec — it's a story. It explains *why* before *how*, shows you the weak approach and the production approach side by side (so you build only the right one), and weaves in two kinds of help:

- **📖 Mandatory reads** — short, required articles placed between chapters, on the ideas the next chapter depends on. Read them. They're how you build your *own* understanding instead of copying steps — which is exactly what you'll be examined on later.
- **💡 Hints** — a nudge when a step is genuinely tricky. A hint points at the thinking, never the answer.

### The checklist is your gate

Every chapter ends in a **✅ Definition of Done** — a short checklist of things you can actually *see or demo* (concept chapters end in **Key takeaways** you can explain instead).

**This checklist is a gate.** You do not move to the next chapter until every box is ticked. No ticking a box you didn't really finish — each item is observable on purpose, so "done" can't be faked. The checklist is your permission to proceed, and the thing that keeps you honest in a self-paced course. If a box won't tick, that's not a blocker to skip — that's where today's work is.

### Keep a learning log (mandatory)

Create a folder called **`learning-log/`** at the root of your project repo, and commit it. Whenever a chapter asks you to *explain* something — every concept chapter, and the "you can explain…" items in build chapters — **write your answer there**, in a file named after the chapter: `learning-log/02-why-relational.md`, `learning-log/14-normalization-vs-denormalization.md`, and so on.

This is **not optional**. Those written answers are:

- the **evidence** that you actually cleared a chapter's gate — not just skimmed it;
- your **rehearsal** for the final viva, where you'll defend these decisions out loud;
- and what your mentor (or the AI reviewer) reads to check you really understand what you built.

Write them like you're convincing a skeptical senior engineer, and use *your own marketplace* in the examples. A one-line answer doesn't count.

### How you're judged

On **understanding**, not just a green demo. At the end you'll walk through how your system works and what would break it — defend a trade-off, trace a request, name where it falls over at ten times the scale. The per-chapter checklists are how you stay ready for that final review.

### When you're stuck

That's the work, not a detour. Read the error. Read the docs. Explain the problem out loud. Try a smaller test. Take a break. *Then* ask — and when you do, lead with "here's what I expected, and here's what actually happened."

## The project — what you're building

A marketplace is a shop with many shops inside it. Independent makers each open a **store** and list their **products** with photos and details. A **customer** browses the whole marketplace — every maker's catalogue in one place — searches, and fills **one cart** that can hold items from several makers at once. When they check out, that single cart becomes **one order per maker**, because each maker fulfils and gets paid for their own items independently. Behind the scenes, the slow work — sending emails, resizing images, totting up nightly payout reports — happens in **background workers**, off the request path, so the app stays fast.

Understand that shape first — one storefront per maker, one cart across makers, one order per maker at checkout — and the rest of the course is filling it in, properly, one production concern at a time.

## The people who use it

- **Vendor (a maker).** Registers, opens a store, lists products with photos and details, sees only their **own** catalogue and orders, and gets paid out for what sells.
- **Customer.** Browses the whole marketplace, searches, fills one cart across multiple makers, checks out once, and tracks their orders.

(There is no admin or moderator role in this build — see *Out of scope*.)

## In scope

- Production project setup: configuration from the environment (validated at startup), secrets out of Git, Postgres and Redis run locally with Docker.
- Real authentication: hashed passwords, access + refresh tokens with rotation, email verification.
- Authorization and **multi-tenant isolation** — a vendor can never read or change another vendor's data.
- A vendor product dashboard, and a public catalogue that stays fast as it grows (no N+1, cursor pagination, deliberate indexing, cached hot reads).
- Product **image uploads** to object storage, resized in the background.
- A **multi-vendor cart and checkout** that splits into one order per vendor, with the price **snapshotted** at purchase, inside a transaction, safe to retry.
- **Background/async processing** with a job queue and workers: order emails, image processing, and scheduled **payout reports** — with retries, idempotency, and failure handling.
- Production hardening: rate limiting, structured logging with request IDs, health/readiness checks, graceful shutdown, and a real deployment running the web app and the worker as separate processes.

## Out of scope

Deliberately **not** building these — so you don't gold-plate the wrong thing:

- **Event-driven architecture** — no pub/sub, no event bus, no CQRS. We do async with a **job queue + workers**; that's the production pattern this course teaches.
- A **real payment processor** — checkout *simulates* payment, and you build everything around it correctly.
- Real-time features (websockets, live chat), a native mobile app, and a full admin panel.

**The flow in one line:** *a maker opens a store and lists products → a customer fills one cart across several makers → checks out once → the order splits per vendor with prices snapshotted → confirmation emails and image processing run in the background → makers see their orders and nightly payout reports.*

## Before you start

This course **starts above CRUD**. You don't need to be an expert — you just need to be comfortable with the handful of basics below. They're the floor the course stands on; everything harder is taught as you go.

| You should be comfortable with | Why you need it |
|---|---|
| Building a basic CRUD REST API on a framework | The course starts here and climbs from it |
| SQL basics — tables, joins, foreign keys | The data model and most queries build on this |
| `async` / `await` in JavaScript/TypeScript | Week 4's background jobs lean on it |
| Everyday Git — commit, branch, `.gitignore` | You'll commit daily and keep secrets out of history |

**Helpful, but taught as needed (don't worry if these are new):** TypeScript beyond the basics, running PostgreSQL and Redis with Docker, and your frontend framework of choice. The course is written for a **Node + TypeScript** backend with **PostgreSQL**, **Redis** (cache + job queue), and **S3-compatible object storage** — Chapter 3 helps you lock in the exact tools. Pick one backend stack and one frontend stack at the start and stay with them.

## How the four weeks are organised

This is a **4-week course at roughly 60 focused hours a week** — around **60 short chapters in 15 modules**. It follows a complexity ladder: it starts at production setup and climbs, one rung at a time, to async processing and deployment.

- **Week 1 — Production foundations** *(Modules 1–3):* set the stack and local environment, do project setup the production way, design the marketplace data model, and validate input properly.
- **Week 2 — Identity, access & isolation** *(Modules 4–7):* real authentication, authorization, the multi-tenant isolation that defines a marketplace, and the vendor experience.
- **Week 3 — Performance & commerce** *(Modules 8–11):* keep the catalogue fast at scale, cache it, handle images, and build the multi-vendor checkout correctly.
- **Week 4 — Async, resilience & ship** *(Modules 12–15):* move slow work into background jobs, run scheduled batches, harden the system, and deploy it.

Work the chapters **in order**, and remember the gate: finish each chapter's checklist before starting the next.

## Course outline

### Week 1 — Production foundations

#### Module 1 — Decisions & setup

| # | Chapter | Done when… |
|---|---------|-----------|
| 2 | [Why a relational database for a marketplace](02-why-relational.md) | You can explain why relational fits orders/ownership here, and the trade-offs |
| 3 | [Choosing your stack & tools](03-choosing-your-stack.md) | Framework, query layer, and libraries chosen — and you can justify each |
| 4 | [Project bootstrap & structure](04-project-bootstrap.md) | The TypeScript app starts with one command; sensible structure in place |
| 5 | [Your local environment with Docker](05-local-environment-docker.md) | Postgres and Redis run locally via Docker; the app connects |
| 6 | [Configuration & secrets from the environment](06-config-and-secrets.md) | Config read from env and validated at startup; `.env` git-ignored; `.env.example` committed |
| 7 | [The health check endpoint](07-health-check.md) | `GET /health` returns 200 with a small JSON body |

#### Module 2 — The data model

| # | Chapter | Done when… |
|---|---------|-----------|
| 8 | [Modeling vendors, stores & customers](08-model-vendors-stores-customers.md) | Tables and relationships for the people and shops |
| 9 | [Modeling products (and JSONB attributes)](09-model-products.md) | Products modeled with relational columns + a JSONB attributes field |
| 10 | [Modeling orders & order items](10-model-orders.md) | Orders and line items modeled, ready for per-vendor splitting |
| 11 | [Foreign keys & constraints](11-foreign-keys-and-constraints.md) | Referential integrity enforced by the schema, not app code |
| 12 | [Migrations: versioning your schema](12-migrations.md) | Schema built from versioned, committed migrations |
| 13 | [Seed data for development](13-seed-data.md) | A seed script loads realistic vendors, products, and orders |
| 14 | [Normalization vs denormalization](14-normalization-vs-denormalization.md) | You can explain the trade-off and where you'll bend it later |

#### Module 3 — Validation & errors

| # | Chapter | Done when… |
|---|---------|-----------|
| 15 | [Validating input at the boundary](15-validation-concept.md) | You can explain why every request is validated before it touches logic |
| 16 | [A request validation layer](16-request-validation.md) | All incoming requests validated against schemas |
| 17 | [Consistent errors & status codes](17-errors-and-status-codes.md) | Every error returns a consistent shape and the correct status |

### Week 2 — Identity, access & multi-tenant isolation

#### Module 4 — Authentication

| # | Chapter | Done when… |
|---|---------|-----------|
| 18 | [Auth & the threat model](18-auth-threat-model.md) | You can explain what you're protecting against and why |
| 19 | [Hashing passwords](19-password-hashing.md) | Passwords stored as salted, slow hashes — never readable |
| 20 | [Register & login](20-register-and-login.md) | Register and login work end to end |
| 21 | [Access tokens (JWT)](21-access-tokens.md) | Login issues a verifiable access token carrying identity |
| 22 | [Refresh tokens, rotation & revocation](22-refresh-tokens.md) | Refresh tokens stored hashed; rotated; revocable |
| 23 | [Email verification (OTP)](23-email-verification.md) | An account stays inactive until its emailed code is confirmed |
| 24 | [The auth middleware](24-auth-middleware.md) | Protected routes reject missing/invalid/expired tokens |

#### Module 5 — Authorization & roles

| # | Chapter | Done when… |
|---|---------|-----------|
| 25 | [Authentication vs authorization](25-authz-concept.md) | You can explain the difference with a concrete example |
| 26 | [Roles & role-based route protection](26-roles-and-route-protection.md) | Vendor-only and customer-only routes reject the wrong role |

#### Module 6 — Multi-tenant isolation

| # | Chapter | Done when… |
|---|---------|-----------|
| 27 | [What multi-tenant isolation means](27-isolation-concept.md) | You can explain the rule every later query must obey |
| 28 | [Scoping every query to its owner](28-query-scoping.md) | Reads and writes are scoped to the logged-in vendor |
| 29 | [The ownership check & IDOR](29-ownership-and-idor.md) | Editing another vendor's resource returns 403/404 |
| 30 | [Proving isolation with an attack test](30-isolation-attack-test.md) | A written test demonstrates the isolation holds |

#### Module 7 — Vendor experience

| # | Chapter | Done when… |
|---|---------|-----------|
| 31 | [Opening a store (vendor onboarding)](31-vendor-onboarding.md) | A vendor can register and open a store |
| 32 | [Creating & editing products](32-product-create-edit.md) | A vendor creates/edits products scoped to their store, validated |
| 33 | [The vendor's product list](33-vendor-product-list.md) | A vendor sees only their own products, paginated |
| 34 | [Frontend: the vendor dashboard](34-frontend-vendor-dashboard.md) | A working dashboard UI over the vendor APIs |

### Week 3 — Performance & commerce

#### Module 8 — Catalogue & performance

| # | Chapter | Done when… |
|---|---------|-----------|
| 35 | [The public catalogue](35-public-catalogue.md) | Customers browse products across all vendors |
| 36 | [The N+1 problem](36-n-plus-1.md) | The catalogue loads in a constant number of queries |
| 37 | [Pagination: cursor vs offset](37-pagination.md) | Catalogue uses cursor pagination; you can explain why not offset |
| 38 | [Indexing & reading query plans](38-indexing.md) | The right indexes exist; an `EXPLAIN` plan proves they're used |
| 39 | [Search & filtering](39-search-and-filtering.md) | Customers can search and filter without a full table scan |

#### Module 9 — Caching

| # | Chapter | Done when… |
|---|---------|-----------|
| 40 | [Caching hot reads with Redis](40-caching.md) | Hot reads served from Redis with a measured speedup |
| 41 | [Cache invalidation (the hard part)](41-cache-invalidation.md) | Cached data is invalidated correctly on change — no stale reads |

#### Module 10 — Images & object storage

| # | Chapter | Done when… |
|---|---------|-----------|
| 42 | [Why images don't belong in the database](42-images-concept.md) | You can explain the three costs of DB-stored images |
| 43 | [Object storage & presigned uploads](43-object-storage-uploads.md) | Images upload directly to object storage via presigned URLs |

#### Module 11 — Cart & checkout

| # | Chapter | Done when… |
|---|---------|-----------|
| 44 | [The multi-vendor cart](44-cart.md) | A cart holds items from multiple vendors |
| 45 | [Splitting checkout into per-vendor orders](45-order-splitting.md) | One checkout creates one order per vendor |
| 46 | [Price snapshots & why](46-price-snapshots.md) | Order lines record the price paid; later price changes don't alter old orders |
| 47 | [The checkout transaction](47-checkout-transaction.md) | Checkout is atomic — orders, stock, and payment commit together or not at all |
| 48 | [Idempotent, safe-to-retry checkout](48-idempotent-checkout.md) | A retried checkout never double-charges or double-orders |

### Week 4 — Async processing, resilience & ship

#### Module 12 — Async processing

| # | Chapter | Done when… |
|---|---------|-----------|
| 49 | [Why async? Job queues, not event-driven](49-why-async.md) | You can explain job-queue async vs inline — and why *not* event-driven |
| 50 | [Setting up the queue & a worker](50-job-queue-setup.md) | A worker process runs jobs off a BullMQ queue, separate from the web app |
| 51 | [First job: order confirmation emails](51-async-emails.md) | Order emails are sent by a background job, not on the request path |
| 52 | [Retries, backoff, failures & idempotency](52-retries-and-idempotency.md) | Failed jobs retry safely; a job run twice does no harm |
| 53 | [Async image processing (thumbnails)](53-async-image-processing.md) | Uploaded images are resized by a background job |

#### Module 13 — Scheduled & batch jobs

| # | Chapter | Done when… |
|---|---------|-----------|
| 54 | [Scheduled jobs (cron-style)](54-scheduled-jobs.md) | A job runs on a schedule reliably |
| 55 | [Vendor payout reports (batch)](55-payout-reports.md) | A scheduled batch job builds each vendor's payout report |

#### Module 14 — Production hardening

| # | Chapter | Done when… |
|---|---------|-----------|
| 56 | [Rate limiting & abuse protection](56-rate-limiting.md) | Sensitive endpoints are rate-limited; you can justify the limits |
| 57 | [Structured logging & request IDs](57-logging-and-request-ids.md) | Logs are structured and traceable by request ID |
| 58 | [Health, readiness & graceful shutdown](58-readiness-and-shutdown.md) | Readiness endpoint and clean shutdown of app and worker |
| 59 | [Security headers & hardening](59-security-hardening.md) | Standard security headers and basic hardening in place |

#### Module 15 — Ship it

| # | Chapter | Done when… |
|---|---------|-----------|
| 60 | [Deploy: app + worker, secrets, HTTPS](60-deploy.md) | Live on a real URL over HTTPS; web and worker run as separate processes |
| — | [Closing — document & prove it](99-closing.md) | Documented, deployed, and you can defend every decision |

## Your working rhythm

Set these habits from Day 1 — you'll be reminded of them in passing as the weeks go on, not nagged about them every chapter:

- **Commit every day**, in small, meaningful commits.
- **Keep secrets out of Git** — `.env` is git-ignored from the very first chapter.
- **Read the error and the docs before reaching for help**, and when you ask, lead with what you expected vs what happened.

---

Ready? Start with **[Chapter 2 — Why a relational database for a marketplace →](02-why-relational.md)**
