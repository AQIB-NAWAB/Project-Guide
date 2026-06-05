# Chapter 13 — Seed data

Your schema now rebuilds from zero with one command (Chapter 12) — which is wonderful, and also means that *every time you run it, you're staring at six empty tables*. You can't build a catalogue page against no products, you can't test a vendor dashboard with no vendor, and you certainly can't *feel* a performance problem (Week 3) on an empty database. You need the schema **full of realistic data** — and, because you now reset your database constantly, you need a repeatable way to refill it that doesn't involve retyping inserts by hand.

That's seeding. And it's the most naturally hands-on chapter yet: you'll write a seed script, run it, reset your database, run it again, and watch your marketplace reappear identically — including a real multi-vendor order that exercises everything you built in Chapters 8–12 at once.

## Where we're headed

By the end you'll have a committed, **idempotent** seed script (`npm run seed`) that fills your fresh schema with a believable marketplace — several makers with stores, a spread of products using their JSONB attributes, some customers, and at least one **multi-vendor checkout that split into per-vendor orders with price snapshots**. It will **refuse to run in production**, and re-running it will always produce the same state.

## Step 1 — Why seed data, and why not the obvious ways

Remember the clean line from Chapter 12: **migrations build structure, seeds fill it with contents.** Keeping them separate is the whole game, so let's see why the tempting shortcuts fail — they're the same kinds of rot you already met with hand-run schema.

- **Inserting test rows by hand in `psql`.** This is what you did for the Chapter 11 lab, and it was fine *for a one-off experiment*. As your everyday data, it rots instantly: the moment you rebuild the database (which is now a single command you'll run all the time), every row is gone and you retype it. Two teammates end up with different "test data." There's no record of what the data even *is*.
- **Putting seed data inside migrations.** Tempting — migrations already run everywhere. But this violates the structure-vs-contents line: a migration runs in **production**, so your fake "Clay & Co" vendor and "Ghost mug" would land in the real database. Migrations create tables; they must never insert your demo rows.

The right approach is a **dedicated seed script** — separate from migrations, committed to Git, run with one command — that you can fire any time you've reset the database.

## Step 2 — Seeds must be idempotent

Here's the first real requirement, and it's where a naive seed script bites. What happens if you run it **twice**?

A naive script that just `INSERT`s its rows will, on the second run, either crash (duplicate `email`, which is `UNIQUE`) or quietly create a *second* Clay & Co — now you have two vendors where you expected one, and your dev data is a mess. A seed script must be **idempotent**: running it once or five times leaves the database in the *same* known state.

For local development, the simplest honest way to get that is **reset-then-insert**: the script clears the relevant tables and repopulates them from scratch, so the end state is identical every time. (A more surgical alternative is *upserts* with deterministic ids — "insert this vendor, or update it if it already exists" — which you'd reach for when you can't wipe the tables. For dev seeding, reset-then-insert is clean and obvious.) Either way, the test is the same: run `npm run seed` twice in a row and the database looks identical after both.

## Step 3 — Make it realistic, and respect the model

Don't seed one product and call it done. Seed a marketplace you'd actually believe — because realistic data is what lets you *see* features now and *feel* problems later (the N+1 queries and pagination of Week 3 only show up when there's real volume to strain them). A good seed has:

- **Several makers**, each an `accounts` row with `role = 'vendor'`, each owning **one store**.
- **A spread of products** that actually uses the JSONB `attributes` from Chapter 9 — a mug with `{ "glaze": "celadon", "capacity_ml": 350 }`, a scarf with `{ "fibre": "merino", "length_cm": 180 }`, a print with dimensions. This proves your hybrid model holds dissimilar items in one table.
- **A few customers** (`role = 'customer'`).
- Prices in **integer cents**, and everything inserted in **foreign-key order** — accounts → stores → products → orders → order_items — or the constraints you built in Chapter 11 will (correctly) reject it.

The seed is also a quiet test of your own schema: if you can't express a believable marketplace in it, the model has a gap.

## Step 4 — Decision: handcrafted fixtures or generated volume?

You actually have two different data needs, and they pull in opposite directions:

1. **Known, named data you can reason about** — "Clay & Co always exists, and its celadon mug is always $20." You need this for development and for writing tests you can assert against.
2. **Volume** — hundreds or thousands of products — to make pagination, indexing, and N+1 problems real in Week 3.

| Approach | Pros | Cons |
|---|---|---|
| **Handcrafted fixtures** (a few named, deterministic rows) | Predictable; referenceable in tests; easy to reason about | Tedious; can't produce volume |
| **Generated data** (a library like Faker) | Realistic volume, fast | Random — you can't assert "the mug costs $20" |

**The course requires a small set of deterministic, handcrafted fixtures as the backbone** — a handful of named vendors, stores, and products you can rely on by name — and *optionally* a bulk generator layered on top for volume when you reach performance work. The backbone is what you build features and tests against; the generated bulk is scenery. Don't let random data become the only data, or you'll have nothing stable to point at.

## Step 5 — The centerpiece: a multi-vendor order with snapshots

Now seed the single most valuable fixture in the whole script — the one that proves the marketplace actually works. Create **one checkout by one customer that bought from two different makers**, and watch it become **two per-vendor orders**, each line **snapshotting** the product's price and title (Chapter 10):

```
Checkout (one customer, one payment)
 ├─ Order → Clay & Co     line: "Celadon mug"  × 1   unit_price_cents 2000   (snapshot)
 └─ Order → WoolWorks     line: "Merino scarf" × 1   unit_price_cents 4500   (snapshot)
```

This one fixture lights up everything: the per-vendor split (Ch 10), the snapshots (Ch 10), the ownership boundaries (Ch 8), the foreign keys (Ch 11). It also gives you real data to build a *vendor's order list* and a *customer's order history* against in the coming weeks. Make this order part of your seed and you've got a working marketplace to develop against from day one.

## Step 6 — Reference data vs test data, and the production guard

A sharp distinction worth internalising: **not all seed data is fake.** There are two kinds:

- **Dev/test fixtures** — your made-up vendors, stores, and products. These exist only to develop and test against and must **never** reach production.
- **Reference data** — essential lookup values an app needs in *every* environment to function (think currencies, country codes, fixed categories). This legitimately belongs in all environments, and is sometimes even seeded via a migration *because* it's structural.

Right now your marketplace is mostly dev fixtures, and that makes the safety rule simple and important: **your seed script must refuse to run in production.** You already have the tool for this — the validated `config` from Chapter 6 knows `NODE_ENV`. Use it as a guard at the very top of the script:

```ts
// at the top of your seed script — fake data must never touch production
import { config } from './shared/config';
if (config.NODE_ENV === 'production') {
  throw new Error('Refusing to run the dev seed against production.');
}
```

That four-line guard is the difference between "reset my dev data" and "I just inserted Ghost Mug into the live store." It's also a satisfying callback: the config module you built in Chapter 6 is now actively protecting you.

## Step 7 — Do it on your project (hands-on)

Build it now, and prove it behaves:

**1. Create the seed script and a command for it.** Add `npm run seed` to your scripts (alongside `migrate` from Chapter 12).

**2. Add the production guard** from Step 6 as the first thing the script does.

**3. Make it idempotent** with reset-then-insert: clear the tables, then insert your fixtures.

**4. Seed a realistic marketplace** — at least two vendors with stores, several varied products using JSONB attributes, a couple of customers — in foreign-key order, money in cents.

**5. Add the multi-vendor order** from Step 5: one customer, two makers, per-vendor orders, snapshotted lines.

**6. Run it and verify** what you built actually landed:

```
npm run seed
psql "$DATABASE_URL" -c "SELECT count(*) FROM products;"     # → your product count
psql "$DATABASE_URL" -c "SELECT name FROM stores;"           # → your makers' stores
-- the centerpiece: two orders, one checkout, with frozen prices
psql "$DATABASE_URL" -c "SELECT o.store_id, oi.title_snapshot, oi.unit_price_cents
                         FROM orders o JOIN order_items oi ON oi.order_id = o.id
                         WHERE o.checkout_id = '<the checkout you seeded>';"
```

**7. Prove repeatability.** Run `npm run seed` **again** and confirm the counts are unchanged (idempotent — no duplicate vendors). Then go the whole way: drop the database, `npm run migrate` to rebuild the empty schema from zero (Chapter 12), `npm run seed` to refill it — and you're back to an identical, working marketplace in two commands. That two-command rebuild is the workflow you'll live in for the rest of the course.

> 💡 **Hint — seed the messy cases, not just the happy ones.** The fixtures you'll be gladdest you made are the awkward ones: a vendor with *zero* products, a product that's `draft` (not published), a customer with no orders, an order with several line items. Happy-path-only seed data hides the bugs that live in the empty and the plural — seed the edges and you'll catch them while building, not in review.

> **📖 Mandatory read — before Chapter 14.** Read your stack's guidance on **database seeding** (most ORMs and migration tools document a seed pattern), and skim one short piece on **idempotent seeding / using Faker for test data** (search *"idempotent database seed"*). *Required: you'll lean on this seeded data through every feature chapter ahead.*

> **Interesting to read.** Realistic seed data isn't just convenience — teams have shipped real bugs precisely because they only ever tested against tidy, tiny datasets, and the bug only appeared at volume or with an empty relationship. Search *"test data realistic edge cases bugs"* to see why "seed the boring and the broken cases" is a discipline, not a chore.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 14:

- [ ] A committed **seed script** runs with **one command** (`npm run seed`), separate from your migrations
- [ ] It is **idempotent** — you ran it twice and the row counts were identical, with no duplicates or errors
- [ ] It **refuses to run when `NODE_ENV` is production** (you can confirm by setting it and watching it abort)
- [ ] It seeds a **realistic marketplace**: multiple vendors with stores, varied products using JSONB attributes, and customers — inserted in foreign-key order, money in cents
- [ ] It includes a **multi-vendor checkout that split into per-vendor orders with price snapshots**, and you queried it to confirm
- [ ] You **rebuilt from zero** (`migrate` then `seed`) and got an identical, working dataset

> **✍️ Log it (mandatory).** In `learning-log/13-seed-data.md` — **decision** first, then **topics**: **(Decision)** Why is seed data a separate, committed script rather than hand inserts or rows inside migrations — give the concrete failure of each shortcut? Why must the seed refuse to run in production, and how did you enforce it? **(Topics)** (1) What does *idempotent* mean for a seed, and exactly how did you make yours idempotent? (2) When do you want handcrafted fixtures versus generated bulk data? (3) What's the difference between dev/test fixtures and reference data? (4) Why does seeding one multi-vendor order with snapshots exercise so much of your model at once?

*All boxes ticked and the log written? Then continue. You now have a marketplace you can rebuild and refill in seconds — the ground every feature stands on. The data module closes with the decision that's been humming under all of it: when is copying data the right thing to do?*

---

Next: you've split data into clean tables all module — but you also deliberately *copied* a price onto an order. When is each right? → **[Chapter 14 — Normalization vs denormalization](14-normalization-vs-denormalization.md)**
