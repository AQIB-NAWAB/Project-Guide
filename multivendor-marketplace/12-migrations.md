# Chapter 12 — Migrations

Last chapter you created your tables **by hand** in `psql` to run those integrity experiments — and you felt how tedious and fragile that was. There was no record of what you ran, no order anyone else could follow, no way to hand a teammate "the schema." If they wanted the same database, they'd retype everything, hope they got the order right, and pray they didn't miss a constraint. That discomfort is the problem this chapter exists to kill.

You've designed and constrained a whole schema — accounts, stores, products, orders, line items, with foreign keys and checks holding it together. But it doesn't *properly* exist anywhere yet. So the real question of this chapter is deceptively small: **how do those tables come to exist — in your database, your teammate's, CI, and production — identically, in the right order, every single time, with one command?**

The answer is migrations, and learning them is as much about *discipline* as tooling. The difficulty here isn't a hard algorithm; it's the operational maturity of treating your schema as versioned code that many machines must agree on. Get the discipline right and rebuilding the entire database is one command; get it wrong and your databases quietly drift apart until bugs appear on one machine and nowhere else.

## Where we're headed

By the end you'll have a `migrations/` folder of **ordered, versioned files** committed to Git, applied by a **runner that records which have run** (so each runs exactly once, everywhere), with a defined way to **undo or roll forward** a change — and one unbreakable rule: an applied migration is never edited.

## Step 1 — The tempting way, and why it rots

The fastest way to create your tables is to open `psql`, paste your `CREATE TABLE` statements, and move on. Or keep a `setup.sql` you run once. It works today — and it rots fast, in concrete ways:

- **It's not reproducible.** A teammate clones the repo and has… what? A pile of SQL to paste by hand, in some order they have to figure out, hoping they didn't miss one. There's no single command that builds the schema correctly.
- **There's no history.** Six weeks in, a column exists and nobody knows when it was added, why, or whether production has it. The database and the code silently disagree.
- **Databases drift.** Your local DB gains a column you added by hand; staging never got it. Now you have a bug that reproduces on one machine and not another — exactly the "works on my machine" problem Docker fixed for *services* in Chapter 5, reappearing for *schema*.
- **There's no rollback.** A change turns out wrong and there's no defined way to undo it.
- **Production is terrifying.** Hand-typing `ALTER TABLE` against a live production database is how outages start.

Migrations fix every one of these by making schema changes *files* — ordered, versioned, committed, and applied by a tool.

## Step 2 — What a migration actually is

A **migration** is a small file describing **one schema change**, with two parts: an **`up`** (apply the change) and, ideally, a **`down`** (undo it). Migrations are **named so they sort in a strict order** — a timestamp or sequence number prefix — and they live in version control beside your code. A **migration runner** looks at which migrations have already been applied, runs the *pending* ones in order, and records each in a small tracking table inside the database itself:

```
migrations/                              schema_migrations   (the DB's own record)
──────────────────────                   ──────────────────────
 0001_create_accounts.sql                 version                 applied_at
 0002_create_profiles.sql                 ────────────────────────────────────
 0003_create_stores.sql                   0001_create_accounts    2026-06-01 …
 0004_create_products.sql                 0002_create_profiles    2026-06-01 …
 0005_create_orders.sql                   0003_create_stores      2026-06-01 …
 0006_create_order_items.sql              …
```

Because the runner records what it applied, running it again is **safe and does nothing** — it only ever applies migrations not already in `schema_migrations`. That's what makes "build the database" a single, repeatable, idempotent command that produces the *same* schema on every machine.

## Step 3 — Up and down: apply and roll back

Each migration knows how to do its change and how to undo it. For one of your tables, the shape is (illustratively — this is schema DDL, the legitimate kind of code a tooling chapter shows):

```sql
-- 0001_create_accounts.sql

-- up: apply the change
CREATE TABLE accounts (
  id         uuid PRIMARY KEY,
  email      text UNIQUE NOT NULL,
  role       text NOT NULL CHECK (role IN ('vendor','customer')),
  created_at timestamptz NOT NULL DEFAULT now()
);

-- down: undo it
DROP TABLE accounts;
```

The `down` matters because it gives a wrong change a **defined exit**. Apply a migration, realise it was a mistake, roll it back, fix it, re-apply. Without a `down`, undoing is improvisation.

> A note on production reality: many teams run **forward-only** in production — instead of rolling a bad change *down*, they write a *new* migration that corrects it and roll *forward*. Rolling back a destructive change (a dropped column) can't recover the data anyway. So `down` is invaluable in development and a safety net to think through, but in production "fix it with the next migration" is often the safer path. Either way, every change is a tracked, ordered file.

## Step 4 — Order follows your foreign keys

Migration order isn't cosmetic — it's forced by the relationships you built in Chapter 11. You cannot create `products` with its `store_id` foreign key before `stores` exists, and you can't create `stores.vendor_id` before `accounts` exists. The database will reject a foreign key to a table that isn't there yet. So the creation order is dictated by the dependency chain:

```
accounts → profiles → stores → products → orders → order_items
   (the same direction the ownership arrows point)
```

This is *why* migrations are numbered and applied in strict sequence. The ordering encodes the dependency graph of your schema — run them out of order and the foreign keys have nothing to point at.

## Step 5 — The golden rule: never edit an applied migration

This is the one rule that makes the whole system trustworthy, so it's worth stating starkly: **once a migration has run anywhere — a teammate's machine, CI, production — it is immutable.** You do not edit it. To change the schema again, you **add a new migration**.

Here's why, concretely. The runner identifies a migration by its name/version and records it as "applied." If you *edit* an already-applied file — say, add a column to `0004_create_products.sql` — the runner sees `0004` is already in `schema_migrations` and **skips it**. So machines that ran the old `0004` keep the old schema; only brand-new databases get your edit. You've manufactured the exact drift migrations exist to prevent. The rule has no exceptions: a schema change is *always* a new file (`0007_add_weight_to_products.sql`), never an edit to history.

> 💡 **Hint — migrations build structure, not data.** Keep a clean line between *schema* and *contents*. Migrations create tables, columns, and constraints — the structure. They should **not** insert your test products or demo accounts; that's **seed data**, and it's the next chapter's job (Chapter 13). Mixing sample rows into migrations means every environment — including production — gets your fake data. Structure in migrations; data in seeds.

## Step 6 — The decision: which migration tool?

You chose a database-access approach back in Chapter 3 (raw SQL, a query builder, or an ORM). Match your migration tool to that choice — and whatever you pick, it must be a *real, repeatable runner*, never hand-run SQL:

- **Raw SQL or a query builder** → a standalone migration runner (for the Node/Postgres world, tools like `node-pg-migrate`, or plain numbered `.sql` files driven by a small runner). You write the SQL migrations yourself, exactly as in Step 3.
- **An ORM (Prisma, Drizzle, etc.)** → its built-in migration tooling (`prisma migrate`, `drizzle-kit`). These often *generate* a migration by diffing your schema definition against the database — convenient, but with one firm caveat below.

| Approach | Pros | Cons |
|---|---|---|
| **Hand-run SQL in psql** | Nothing to set up | Not reproducible, no history, no tracking — **forbidden** |
| **Standalone SQL runner** | Full control; explicit SQL; tool-agnostic | You write every change by hand |
| **ORM-integrated migrations** | Generates migrations from your schema; less boilerplate | Can generate *destructive* steps you didn't intend |

**The course requires a real migration tool that fits your Chapter 3 stack** — not hand-run SQL. And one rule binds the convenient ORM path: **always read a generated migration before you commit it.** Auto-generation can decide to drop and recreate a column (losing data) or reorder something destructively; the generated file is a *draft you review*, not a black box you trust. Read it, understand each statement, then commit.

## Step 7 — Do it on your project, and prove it works (hands-on)

Enough theory — build the real thing now. Throw away the by-hand tables from Chapter 11 and recreate your whole schema as migrations. Work through this in order; each step is something you *do* and then *check*.

**1. Start clean.** Drop the throwaway tables you made by hand last chapter (or just drop and recreate the database) so you're migrating into an empty schema. You want the migrations to be the *only* thing that ever built these tables.

**2. Add your migration tool and a script for it.** Install the tool that matches your Chapter 3 choice, and make running it a first-class command, the way you made `dev` and `build` in Chapter 4:

```jsonc
// package.json (scripts) — names illustrative; use your tool's commands
"scripts": {
  "migrate":        "...run all pending migrations...",
  "migrate:down":   "...roll back the last migration...",
  "migrate:make":   "...create a new, empty, timestamped migration..."
}
```

**3. Write and run your first migration.** Create `0001_create_accounts`, put your constrained `accounts` table in its `up` and the reverse in its `down` (Step 3), then run `npm run migrate`. Now check it actually happened:

```
psql "$DATABASE_URL" -c "\dt"                              # → accounts appears
psql "$DATABASE_URL" -c "SELECT * FROM schema_migrations;" # → 0001 is recorded
```

**4. Build the rest — in dependency order.** Add a migration per table, in the order your foreign keys demand (Step 4): profiles → stores → products → orders → order_items, each carrying the foreign keys (with your `ON DELETE` choices) and constraints you decided in Chapter 11. Run `migrate` after each, or write them all and run once. When `\dt` shows all six tables, your *designed* schema is now a *real* one — with zero hand-typed SQL in `psql`.

**5. Prove rollback works.** Roll back the last migration and watch it reverse, then re-apply it:

```
npm run migrate:down     # → order_items is dropped; schema_migrations no longer lists it
npm run migrate          # → order_items is back. A change has a defined undo.
```

**6. Prove the golden rule — by breaking it on purpose.** Open an already-applied migration (say `0004_create_products`), add a throwaway column to its `up`, and run `npm run migrate` again. Watch what happens: **nothing.** The runner sees `0004` is already recorded and skips it — your edit never reaches the database, while a brand-new database *would* get it. You've just watched drift being born. Now **revert your edit** and, if you genuinely wanted that column, add it as a *new* migration (`0007_add_...`) — the correct way.

**7. The payoff — rebuild from zero.** Drop the entire database, recreate the empty database, and run one command:

```
npm run migrate          # → the whole marketplace schema, rebuilt identically, in order
```

That single repeatable command is the entire point of the chapter — the thing you couldn't do by hand in Chapter 11. To close the loop, re-run a couple of your Chapter 11 experiments (insert a product with a fake `store_id`; try to delete a sold product) against this freshly-migrated schema and confirm the constraints still bite. They do — because the migrations carry them.

Migrations also run **before the app serves traffic** — locally before `npm run dev`, and as a step in your deploy pipeline before new code goes live (Chapter 60).

> **📖 Mandatory read — before Chapter 13.** Read your chosen tool's **migrations guide** (Prisma Migrate, Drizzle Kit, or your runner's docs), and one general piece on **database migration best practices** (search *"database migrations best practices never edit"*). *Required: you're about to run real migrations, and the "never edit an applied migration" discipline is assumed from here on.*

> **Interesting to read.** Schema migrations have taken down large sites — a single `ALTER TABLE` that locks a huge table while it rewrites can freeze every query against it for minutes. It's why big teams use special "online" migration techniques for large tables. Search *"migration locked table outage"* — a reminder that on real data, *how* a migration runs matters as much as *what* it does.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 13:

- [ ] Migrations live in **version control** as **ordered, versioned files** (timestamp or sequence prefix), one per table, in dependency order
- [ ] A `migrate` script applies pending migrations and records them; **running it twice applies nothing new** (you checked `schema_migrations`)
- [ ] You **rolled back** the last migration and **re-applied** it, and saw the table disappear and return
- [ ] You **edited an applied migration on purpose and watched the runner skip it** — so you understand first-hand why it's never done; you reverted the edit
- [ ] You **dropped the database and rebuilt the entire schema from zero with one command**, and re-confirmed a Chapter 11 constraint still rejects bad data
- [ ] No hand-typed SQL built these tables — the migrations are the single source of truth
- [ ] Migrations contain **structure only** — no seed or test data

> **✍️ Log it (mandatory).** In `learning-log/12-migrations.md` — **decision** first, then **topics**: **(Decision)** Having now built the schema both ways, why are migrations better than the by-hand approach you used in Chapter 11 — name the concrete failures hand-run schema causes? When you edited an applied migration and re-ran the runner, what happened, and why does that prove you must never edit one? **(Topics)** (1) What does the runner's tracking table give you, and why does that make re-running safe? (2) What are `up` and `down`, and what is the "roll forward" alternative to `down` in production? (3) Why is migration *order* forced by your foreign keys — give an example from your schema? (4) Why must an ORM-generated migration still be read before committing?

*All boxes ticked and the log written? Then continue. The database can now be built from zero on any machine, in order, on demand. Next you fill that empty structure with realistic data to develop against.*

---

Next: an empty schema is hard to build features against — you need realistic vendors, stores, and products to see and query. → **[Chapter 13 — Seed data](13-seed-data.md)**
