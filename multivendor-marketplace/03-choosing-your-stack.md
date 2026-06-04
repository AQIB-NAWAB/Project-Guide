# Chapter 3 — Choosing your stack & tools

The database is decided. Now you assemble the rest of the toolbox: the web framework, how you talk to Postgres, how you validate input, and the supporting cast (Redis, a job queue, object storage). This is still a *decision* chapter — you won't install anything yet — because the goal is the same as Chapter 2: not just to pick tools, but to **be able to defend every pick** when someone asks "why this and not that?"

There's a quiet trap here, and it's worth naming before you start. The tempting way to choose a stack is *"whatever the last tutorial used"* or *"whatever's trending on Twitter this month."* That's how people end up running a tool they can't explain, can't debug, and can't justify in an interview. The professional way is the opposite: pick **boring, proven tools you can reason about**, and **write down why**. Newer is not better; *understood* is better.

## The point of this chapter

By the end you'll have chosen — and recorded, with reasons — every major piece of your stack, on top of the Node + TypeScript + PostgreSQL + Redis base this course assumes. You'll make four real decisions (framework, database access, validation, supporting services) and learn the *axis* each one turns on, so the choice is yours and you own it.

## Decision 1 — The language: why TypeScript, not plain JavaScript

You're building on Node, and you have a fork in the road: plain JavaScript, or **TypeScript** (JavaScript with a type system layered on top, checked before your code runs).

The tempting argument for plain JS is speed: no build step, no type annotations, just write and run. For a weekend script, fine. For a marketplace with money, orders, and many moving parts, it disappoints fast — the bugs that plain JS lets through (a `vendorId` that's `undefined`, an order total that's a string `"19.99"` instead of a number, a function called with the wrong shape) are exactly the bugs that cost you in production, and they show up at runtime, in front of a user, instead of at compile time, in front of you.

TypeScript catches a whole class of those before the code ever runs. For example, the kind of mistake plain JavaScript ships and TypeScript refuses:

```ts
function chargeOrder(totalCents: number) { /* … */ }

chargeOrder("1999");        // ❌ TS error: 'string' is not assignable to 'number'
chargeOrder(order.total);   // ❌ TS error if order.total might be undefined
```

In plain JavaScript both calls run happily — and you find out when a customer is charged the string `"1999"` or a `NaN`. TypeScript moves that discovery from production to your editor. And — just as important on a 60-chapter project — it makes the code *self-documenting*: the types tell the next developer (often future-you) what shape a function expects. That's why this course uses TypeScript, and why most serious Node teams do. You'll lean on it from Chapter 5 onward.

> **One thing to file away now:** TypeScript's types exist only at *compile* time. They are **erased** before your code runs — at runtime there are no types at all. That sounds like a footnote; it's actually the reason Decision 3 (validation) exists. Hold that thought.

## Decision 2 — The web framework: minimal vs batteries-included

Every Node web framework sits somewhere on one axis: **how much it does for you out of the box.**

- **Minimal / unopinionated** (e.g. **Express**) — gives you routing and middleware and gets out of the way. You assemble everything else yourself. Maximum freedom, maximum decisions, and easy to make a mess if you're undisciplined.
- **Modern-but-lean** (e.g. **Fastify**) — similar shape to Express, but faster, with first-class TypeScript support and built-in schema validation hooks. A strong middle ground.
- **Batteries-included / opinionated** (e.g. **NestJS**) — gives you a full structure: modules, dependency injection, conventions for everything. Less to decide, a steeper learning curve, and the framework's opinions become your app's opinions.

You can *see* the axis in how you declare a route — the same endpoint, less vs more structure:

```ts
// Minimal (Express / Fastify) — you wire each piece yourself, explicitly
app.get('/products', listProducts);

// Batteries-included (NestJS) — decorators and conventions do the wiring
@Controller('products')
class ProductsController {
  @Get() list() { /* … */ }
}
```

Neither is "better" — the top gives you fewer surprises, the bottom gives you more structure for free. The question is only how much magic you want between your code and the request.

There's no universally right answer — there's a right answer *for your team and project*. A solo learner who wants to see how the pieces fit might pick Express or Fastify; a team that values consistency on a large codebase might pick NestJS. For this course, **Fastify is a clean default** (fast, typed, not too magical), and **Express is perfectly fine** if it's what you know. The point isn't the brand — it's that you can say *why* you landed where you did on that "how much does it do for me?" axis.

> 💡 **Hint — how to choose on this axis.** Ask: *"How much of this framework's magic will I have to debug at 2 a.m.?"* The more a framework does for you, the more you must understand what it's doing when it breaks. Pick the most help you can still fully explain.

## Decision 3 — How you talk to the database (the one that bites juniors)

This is the most consequential pick in the chapter, because the wrong instinct here causes pain for the whole project. There's a spectrum:

| Approach | What it is | The trade-off |
|---|---|---|
| **Raw SQL** | You write every query as a SQL string | Total control, zero magic — but verbose, easy to make mistakes, and no type safety between your code and the result |
| **Query builder** (e.g. Drizzle, Knex) | You build SQL with typed function calls | Close to SQL, typed, predictable — you still think in queries |
| **ORM** (e.g. Prisma, TypeORM) | You work with objects; it generates the SQL | Fast to write, great DX, typed models — but it hides the SQL, which can hide performance problems |

The same query — *"products for this vendor"* — looks like this in each style (illustration; you'll write the real ones later):

```ts
// Raw SQL — you write the string and map the result yourself
const rows = await db.query(
  'SELECT id, title, price_cents FROM products WHERE vendor_id = $1',
  [vendorId],
);

// Query builder (Drizzle) — typed function calls, still clearly SQL-shaped
const rows = await db.select().from(products).where(eq(products.vendorId, vendorId));

// ORM (Prisma) — you ask for objects; it writes the SQL for you
const rows = await prisma.product.findMany({ where: { vendorId } });
```

Read top to bottom and you can watch the dial turn: most *control and visibility* at the top, most *convenience and abstraction* at the bottom. Neither end is "right" — but the extremes are where people get hurt:

Two opposite temptations, and both disappoint:

- **Hand-roll everything in raw SQL** because "real engineers write SQL." You'll be productive for a day, then drown in string-concatenated queries with no type safety, and you'll reinvent migrations, connection pooling, and result mapping badly.
- **Reach for a full ORM and let it do everything**, treating the database as invisible. This is faster to write — until the ORM quietly issues *one query per item in a loop* (the **N+1 problem** you'll meet in Chapter 36) and your catalogue page fires 200 queries while you wonder why it's slow. The ORM didn't make the problem; it made it *easy not to notice*.

The professional middle is: **use a typed ORM or query builder for the productivity and type-safety — but keep your SQL literacy sharp enough to know what it's doing underneath.** The tool writes the boilerplate; *you* stay responsible for the queries it generates. For this course, **Prisma** (excellent DX, typed client, first-class migrations) or **Drizzle** (lighter, SQL-first, typed) are both strong picks. Whichever you choose, the rule is the same: never let the abstraction stop you from being able to see and reason about the actual SQL.

> **📖 Mandatory read — before Chapter 8.** Read one balanced comparison of **ORM vs query builder vs raw SQL** (search: *"ORM vs query builder vs raw SQL trade-offs"*), and skim your chosen tool's "getting started" page so you know its shape. *Required: the data-model module (Chapter 8) builds your schema with this tool; you need to know what it does for you and what it doesn't.*

## Decision 4 — Validating input: why a schema library, not just types

Remember the thought you filed away: **TypeScript's types vanish at runtime.** Here's why that matters now.

A customer's browser (or a malicious script) sends your API a JSON body. TypeScript can *say* a request body is `{ email: string, price: number }`, but that's a compile-time promise about *your* code — it does nothing to the actual bytes arriving over the network. At runtime, that body could be anything: a missing field, a price of `"free"`, a 10 MB string where you expected a name. If you trust the TypeScript type and skip a real check, you've built your validation on a guarantee that doesn't exist at the moment you need it.

So the production answer is a **runtime schema-validation library** — **zod** is the common choice in TypeScript — that checks the *actual* incoming data against a schema at the boundary, and (bonus) infers the TypeScript type from that same schema, so your compile-time and runtime truths can't drift apart. Concretely, the difference looks like this:

```ts
// ❌ Tempting: trust a TypeScript type — but it's ERASED before the request arrives,
//    so nothing checks the real bytes; `priceCents: "free"` sails straight through.
type CreateProductBody = { title: string; priceCents: number };

// ✅ Real: a zod schema inspects the ACTUAL incoming data at the boundary…
const CreateProduct = z.object({
  title: z.string().min(1),
  priceCents: z.number().int().positive(),   // rejects "free", -5, 9.99, etc.
});

// …and the TypeScript type is INFERRED from it — one source of truth, can't drift:
type CreateProduct = z.infer<typeof CreateProduct>;
```

The `type` does nothing at the moment a malformed body arrives; the `z.object(...)` is what actually rejects it. You'll build this validation layer properly in Module 3 (Chapters 15–17); for now, the decision is simply: **validation is a real runtime step, and you'll use a schema library for it — not the type system.**

## Decision 5 — The supporting cast

Three more tools are already in your stack for reasons you'll meet later — pick them now so the foundation is whole:

- **Redis** — an in-memory data store. It earns its place by doing **double duty**: it's your **cache** (Module 9, for hot reads) *and* the backbone of your **job queue** (Module 12). One service, two jobs — fewer moving parts in production.
- **A job queue — BullMQ** (Redis-backed) — runs slow work (emails, image resizing, payout reports) in a separate worker process instead of on the request. This is your async story for Week 4, and it's a **job queue, not an event bus** — keep that distinction (it's why event-driven architecture is out of scope). The shape is simple:

  ```
  HTTP request ──▶ add a job to the queue ──▶ respond fast ✅
                            │
                       Redis (queue)
                            │
                    Worker process ──▶ does the slow work later
                                        (send email · resize image · build report)
  ```

  The request doesn't wait for the slow work; it hands it off and returns. That's the whole point of Week 4.
- **Object storage — S3-compatible** (AWS S3 in production; **MinIO** runs an S3-compatible store locally) — holds product images, served by URL, never stuffed into the database (you'll see exactly why in Chapter 42).
- A **structured logger** (e.g. **pino**) and a **test runner** (e.g. **Vitest** or **Jest**) round it out — you'll wire logging in Module 14.

You don't install these today; you've just chosen them and you know what each is *for*. Here's the whole stack at a glance — the dependencies you'll end up pulling in over the course (illustration; exact names follow your framework/ORM picks):

```jsonc
// the shape of package.json dependencies for this stack
{
  "dependencies": {
    "fastify":             "...",   // web framework        (or express / @nestjs/*)
    "@prisma/client":      "...",   // database access      (or drizzle-orm)
    "zod":                 "...",   // runtime validation
    "ioredis":             "...",   // Redis client         (cache + queue connection)
    "bullmq":              "...",   // job queue            (Redis-backed)
    "@aws-sdk/client-s3":  "...",   // object storage       (S3-compatible)
    "pino":                "..."    // structured logging
  }
}
```

Each one arrives in the chapter that needs it — this is just the map of where you're headed.

## Write your stack down — an ADR

One habit that separates a thrown-together project from an engineered one: **record your decisions.** Professionals keep short **Architecture Decision Records (ADRs)** — a paragraph each: *what* you chose, *why*, and *what you traded away*. It saves the next person (and future-you) from re-litigating settled choices, and it's gold in a viva.

> **Make it yours.** Your first ADR is a great first build-in-public post: *"My production Node stack for a marketplace — and why I picked each piece."* Write the honest version, including the choices you went back and forth on.

## Key takeaways — write them up (mandatory)

This is a decision chapter, so the gate is your reasoning, captured where it can be reviewed.

> **✍️ Where your answers go.** In your repo's **`learning-log/`** folder, create **`learning-log/03-choosing-your-stack.md`** and answer the prompts below. This doubles as your stack ADR — keep it; you'll reference it for the rest of the course. **Mandatory**, reviewed by your mentor/AI, and your viva rehearsal.

### Part A — The decision

1. **Your stack, recorded.** List the tool you chose for each slot — web framework, database access (ORM/query builder), validation library — plus the given base (Node + TypeScript, PostgreSQL, Redis, BullMQ, S3-compatible storage). One sentence each on *why* and *what you traded away*.
2. **The pick you debated most.** Which decision was closest, and what tipped it? (If none felt close, you may not have considered the alternatives hard enough — go back to that one.)

### Part B — The topics you just learned

3. **TypeScript vs JavaScript.** Give one concrete bug in *our* marketplace that TypeScript would catch before runtime that plain JavaScript would let reach a user.
4. **The framework axis.** Explain the "minimal ↔ batteries-included" axis in your own words, and say where your chosen framework sits and why that suits this project.
5. **Database access spectrum.** Explain raw SQL vs query builder vs ORM. Then describe the specific danger of leaning on a full ORM without watching the SQL — name the problem it tends to hide.
6. **Runtime validation.** Explain *why* TypeScript types are not enough to validate an incoming request, in terms of when types exist. What does a schema library (zod) give you that the type system cannot?
7. **Redis's double duty.** Name the two distinct jobs Redis does in this stack, and which later module uses each.

*Gate: don't start Chapter 4 until all seven answers are written and committed in `learning-log/03-choosing-your-stack.md`, and your stack is recorded as an ADR you'd be happy to defend.*

---

Next: choices made — now scaffold the app those tools will live in. → **[Chapter 4 — Project bootstrap & structure](04-project-bootstrap.md)**
