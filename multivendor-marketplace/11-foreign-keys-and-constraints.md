# Chapter 11 — Foreign keys and constraints

Across the last three chapters you drew a lot of arrows: a store belongs to a vendor, a product belongs to a store, an order line points at an order and a product. So far those arrows are just *intentions*. To the database, a `store_id` column holding a UUID is nothing more than text that happens to look like another row's id. Right now, nothing stops you inserting a product whose `store_id` points at a store that was never created, and nothing stops a maker deleting a product a customer **already paid for** — silently destroying the order's history.

This is the chapter where the marketplace's integrity stops being a diagram and becomes something the database *refuses to violate*. And it's a hands-on chapter: by the end you won't just have read about foreign keys, you'll have built a slice of your real schema and **tried to break it with your own hands** — watching the database reject a fake order, block a dangerous delete, and protect a customer's purchase. That's the part that makes the rules stick.

## Where we're headed

By the end, every relationship in your schema is a real **foreign key** the database enforces; every column that should never be empty, duplicated, or out of range carries the right **constraint** (`NOT NULL`, `UNIQUE`, `CHECK`, enum); every relationship has a **deliberately chosen delete behaviour**; and you've *proven* all of it by running experiments against your own database.

## Step 1 — Why the database must enforce this, not your code

The tempting position: "my application code only ever inserts a valid `store_id`, so I don't need the database to check." Here's why that fails in a real marketplace.

Your application is not the only thing that writes to the database, and it is not bug-free. A migration script, a one-off fix you run by hand in `psql`, a future second service (the worker you'll build in Week 4 writes too), a race between two requests, or simply a coding mistake in one vendor-facing endpoint — any of these can write a row your validation didn't guard. The day a product is saved with a `store_id` that matches no store, every catalogue query that joins products to stores either crashes or silently drops that product, and you find out in production with no idea how the data got that way.

A **foreign key** makes that row *impossible to write in the first place*. It tells the database: this column must point at a real row in that table, or the write is rejected. That guarantee is called **referential integrity** — references actually refer to something — and it belongs in the database because the database is the one gate *every* write passes through, no matter which code, script, or person did the writing.

> The principle: validate in the app to give the user a friendly error; *enforce* in the database to make the bad state impossible. They're not redundant — the app check is courteous, the constraint is load-bearing.

## Step 2 — Foreign keys: making a reference real

A foreign key is a column declared to reference another table's primary key. Once declared, the database enforces two things automatically: you can't insert a row whose foreign key points nowhere, and you can't delete a row something still points at — *unless* you've told it what to do instead (Step 3). Every `_id` column you've written becomes a real reference:

```
products.store_id        → REFERENCES stores(id)
stores.vendor_id         → REFERENCES accounts(id)
orders.store_id          → REFERENCES stores(id)
orders.customer_id       → REFERENCES accounts(id)
order_items.order_id     → REFERENCES orders(id)
order_items.product_id   → REFERENCES products(id)
```

With these in place, an order line can never reference a missing order, and a product can never belong to a store that doesn't exist. The arrows from your ERDs become rules.

## Step 3 — The decision: what happens on delete?

This is the chapter's hard part, and the answer is genuinely different for each relationship. When you delete a row, the database needs to know what to do with rows that reference it. Three behaviours matter:

- **`ON DELETE CASCADE`** — delete the children too. Delete a store, its products go with it.
- **`ON DELETE RESTRICT`** — *refuse* the delete while children still exist. You can't remove the thing until you've dealt with what depends on it.
- **`ON DELETE SET NULL`** — keep the child, but null out its reference.

Choosing wrong is dangerous in opposite directions: a careless `CASCADE` erases data you needed; a careless `RESTRICT` leaves you unable to delete anything. Decide per relationship by asking *"does the child still mean anything once the parent is gone?"*

Now walk your **actual marketplace relationships** through that question — and notice the last one is the whole point of the chapter:

- **A store is deleted → its products?** A product is meaningless without its store. **`CASCADE`** — when *Clay & Co* closes, its listings go with it.
- **An order is deleted → its line items?** A line item is wholly part of its order. **`CASCADE`**.
- **A maker deletes a product → the order line items that already sold it?** **Stop.** This is the one that matters, and it's where Chapter 10 pays off. An order line is a **historical record of a sale**, and it already *snapshotted* the price and title — so it doesn't need the product to exist anymore. You must **never `CASCADE`** here: cascading would delete *past sales* every time a maker tidies their shop, destroying financial history a customer has a receipt for. Use **`RESTRICT`** (a product that has ever sold can't be hard-deleted) or **`SET NULL`** on the line's `product_id` (the sale survives, losing only its now-irrelevant traceability link). The order is preserved either way.

| Relationship | On delete | Why |
|---|---|---|
| store → products | `CASCADE` | A product is nothing without its store |
| order → order_items | `CASCADE` | Line items are part of the order |
| account → store | `CASCADE` | Removing a vendor removes their store |
| **product → order_items** | **`RESTRICT` / `SET NULL`** | **An order is history; never delete a past sale** |

The takeaway: `CASCADE` is right when the child is *owned by and meaningless without* the parent; `RESTRICT`/`SET NULL` is right when the child is a **record that must outlive** the parent. In a marketplace, orders outlive products — that single insight is the difference between honest books and a system that eats its own history.

> 💡 **Hint — "soft delete" is usually the real answer.** Makers *want* to remove old listings, but you *can't* hard-delete a product that's been sold. The professional resolution is a **soft delete**: a `deleted_at` (or `status = 'archived'`) column that hides the product from the catalogue without removing the row. It vanishes from the store but stays intact for every order that references it. When a hard delete fights your history, a soft delete is what you actually wanted.

## Step 4 — The other constraints that keep data honest

Foreign keys guard *relationships*; these guard *values*. You've been writing most of them already — here they're named and made deliberate:

- **`NOT NULL`** — the column must have a value. An order with no `store_id` is nonsense; forbid it.
- **`UNIQUE`** — no duplicates. `accounts.email`, `stores.slug`, and the `UNIQUE` foreign keys that made your one-to-one relationships in Chapter 8.
- **`CHECK`** — a value must satisfy a rule. `price_cents >= 0`, `quantity >= 1` — the database itself rejecting impossible numbers, so a bug can't persist a negative price.
- **Enums (or `CHECK ... IN (...)`)** — a column may only hold a value from a fixed set: `role`, `status`. A typo'd `'shippd'` is rejected at the door.

## Step 5 — Prove it: build a slice and try to break it (hands-on)

Reading that foreign keys protect you is one thing; watching one slam the door on a bad row is another. So let's make it real. You've had **Postgres running since Chapter 5** — now you'll create a slice of your schema and attack it.

> ⚠️ **This is a throwaway lab.** You're going to create these tables *by hand* in `psql` just to experiment. Next chapter you'll build the real, repeatable schema with migrations — so don't get attached to what you make here. In fact, pay attention to how *tedious and un-repeatable* doing it by hand feels: that discomfort is the exact problem Chapter 12 solves.

**First, set the stage** — assemble the constrained DDL you've designed across Chapters 8–11 (the tables, their foreign keys with the `ON DELETE` choices above, and the `NOT NULL`/`UNIQUE`/`CHECK`/enum constraints) and create the tables in your dev database. Then insert one tiny, real marketplace scenario: a vendor account, a store they own, a product in that store, a customer, an order *for that store*, and one order line that **snapshots** the product's price and title. (Write the inserts yourself — it's your data and your schema; the experiments below are the throwaway probes.)

Now attack it. Run each of these and watch what the database does:

**Experiment 1 — a reference to nothing.** Try to create a product in a store that doesn't exist:

```sql
INSERT INTO products (id, store_id, title, price_cents, currency, status)
VALUES (gen_random_uuid(), gen_random_uuid(), 'Ghost mug', 1999, 'USD', 'draft');
-- ❌ ERROR: insert or update on "products" violates foreign key constraint
```

The foreign key refused a product with no real owner. That row *cannot exist*.

**Experiment 2 — the one that matters: delete a sold product.** As the maker, try to delete the product your customer already bought:

```sql
DELETE FROM products WHERE id = '<the product in the order>';
-- ❌ ERROR: update or delete on "products" violates foreign key constraint on "order_items"
```

*Clay & Co cannot erase a product a customer already paid for.* The order's history is protected by the database — not by hoping your application code remembers to check.

**Experiment 3 — change the price, inspect the order.** Now confirm the snapshot from Chapter 10 actually holds:

```sql
UPDATE products  SET price_cents = 2500 WHERE id = '<the sold product>';
SELECT unit_price_cents FROM order_items WHERE product_id = '<the sold product>';
-- → still 1999.  The snapshot froze the sale; the past is unchanged.
```

This is Chapters 10 and 11 working *together*: the snapshot froze the price, the foreign key protects the record.

**Experiment 4 — impossible numbers.** Try a negative price and a zero quantity:

```sql
INSERT INTO products (...) VALUES (..., -500, ...);   -- ❌ violates check constraint price_cents >= 0
UPDATE order_items SET quantity = 0 WHERE ...;        -- ❌ violates check constraint quantity >= 1
```

**Experiment 5 — cascade where it's right.** Create a *second* store that has products but **no orders**, then delete it:

```sql
DELETE FROM stores WHERE id = '<the store with products but no orders>';
-- ✅ succeeds — and its products vanish with it (CASCADE). The child was meaningless without the parent.
```

You've now watched every rule fire: a fake reference rejected, a sold product protected, a snapshot held firm, impossible numbers refused, and a clean cascade. The guarantees aren't theory — they're the database actively defending your marketplace. *(And remember the tedium of typing all this by hand — Chapter 12 is about never doing that again.)*

> **📖 Mandatory read — before Chapter 12.** Read PostgreSQL's documentation on **constraints** (search *"postgres foreign key on delete"* and *"postgres check unique not null"*), focusing on the `ON DELETE` options. *Required: Chapter 12 turns this schema into real migrations and you'll be writing these constraints for keeps.*

> **Interesting to read.** A surprising number of real data-loss incidents trace to a missing foreign key or an `ON DELETE CASCADE` someone added without thinking — an admin deletes one row and a cascade silently takes thousands of related rows with it. Search *"accidental cascade delete production"*: the clearest argument for treating every `ON DELETE` as a real decision, exactly as you just did.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 12:

- [ ] Every `_id` reference column is a real **foreign key**; a bad reference is **rejected** by the database (you saw Experiment 1 fail)
- [ ] Each foreign key has a **deliberately chosen `ON DELETE`**, and you can justify each one
- [ ] You **proved a sold product cannot be hard-deleted** (Experiment 2) and that **changing its price left the order's snapshot unchanged** (Experiment 3)
- [ ] A store with products but no orders **cascades** its products on delete (Experiment 5)
- [ ] `CHECK` constraints **rejected** a negative price and a zero quantity (Experiment 4); `email`/`slug` are `UNIQUE`; `status`/`role` are limited to fixed sets
- [ ] You can state in one sentence why these guarantees live in the **database**, not only in app code

> **✍️ Log it (mandatory).** In `learning-log/11-foreign-keys-and-constraints.md` — **decision** first, then **topics**: **(Decision)** Why enforce referential integrity in the database rather than trusting application code — give a concrete way bad data gets in without it? For a product that has been sold, why must its `ON DELETE` *not* be `CASCADE`, and what did you observe when you tried to delete one? **(Topics)** (1) What two writes does a foreign key make impossible? (2) Explain `CASCADE` vs `RESTRICT` vs `SET NULL`, giving the relationship in *your* schema suited to each. (3) Why is a *soft delete* the usual answer for a product a maker wants to remove? (4) From your experiments: which constraint protected the customer's purchase, and which protected the order's *price*?

*All boxes ticked and the log written? Then continue. Your schema is now self-defending — bad data can't exist even if your code slips. But you built it by hand, which doesn't scale to a teammate or to production. Next you fix exactly that.*

---

Next: you created those tables by hand to experiment — now make them real, versioned, and rebuildable by anyone with one command. → **[Chapter 12 — Migrations](12-migrations.md)**
