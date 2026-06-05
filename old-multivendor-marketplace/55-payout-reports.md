# Chapter 55 — Payout reports

A marketplace's whole reason to exist is that vendors get *paid*. You split each checkout into per-vendor orders precisely so each maker's sales could be tracked and settled separately (Chapter 45). Now you build the recurring **batch job** that turns those orders into **payout reports** — for each vendor, over each period, how much they earned. It combines everything from Week 4: it's a **scheduled** job (Chapter 54), it does **batch** work across many vendors, and it must be efficient and idempotent.

It's also the chapter where the per-vendor order model, the price snapshots, and the isolation you built all pay off at once — a payout report is only correct *because* orders are split per vendor and record what was actually paid.

## Where we're headed

By the end a scheduled batch job computes each vendor's payout for a period (summing their completed orders' snapshotted totals), stores the reports, and exposes each vendor's own reports through a scoped endpoint — built to scale across many vendors without N+1.

## Step 1 — What a payout report is

For one vendor, over one period (say, last week): the sum of what they earned from **completed** orders, minus any marketplace fee, equals their payout. The inputs are all things you've already modelled correctly:

- **Per-vendor orders** (Chapter 45) — a vendor's earnings are exactly their orders' totals, already separated from other vendors'.
- **Snapshotted totals** (Chapter 46) — each order's `total_cents` is what was *actually paid*, frozen, so summing them is honest even if products were later repriced.
- **Order status** (Chapter 10) — only count orders in a payable state (e.g. `delivered`/`completed`), not pending or cancelled ones.

```
payout(vendor, period) = Σ order.total_cents
   WHERE order.store_id = vendor's store
     AND order.status = 'completed'
     AND order.created_at IN [period]
   (− marketplace fee, if any)
```

This is why the earlier modelling mattered: a payout report is a straightforward aggregation *because* the data was shaped right. Had you used one shared order or live prices, this would be a mess.

## Step 2 — Batch processing: many vendors, efficiently

A payout run processes *every* vendor, which makes efficiency a real concern — this is **batch processing**. The naive approach is the N+1 trap (Chapter 36) at job scale: loop over vendors, run a query per vendor. With thousands of vendors that's thousands of queries.

Do it set-based instead: a single aggregation query computes *all* vendors' payouts at once — `GROUP BY store_id` over the period's completed orders gives every vendor's total in one pass:

```sql
-- one query, all vendors' payouts for the period
SELECT store_id, SUM(total_cents) AS payout_cents, COUNT(*) AS order_count
FROM orders
WHERE status = 'completed' AND created_at >= $periodStart AND created_at < $periodEnd
GROUP BY store_id;
```

Let the database aggregate — it's what it's best at — and write the results. This is the batch-processing mindset: process the *set*, not each element in a loop.

## Step 3 — Idempotency and the period key

Per Chapter 52/54, the payout job must be idempotent — a retry or accidental double-run must not pay anyone twice (the highest-stakes double-run in the whole app). The natural idempotency key is **`(vendor, period)`**: a payout report for a given vendor and period should exist exactly once. Enforce it with a `UNIQUE (store_id, period)` constraint on the reports table; a re-run that tries to insert a duplicate is rejected (or upserts the same figure), so "ran twice = ran once." Money jobs are where idempotency stops being good practice and becomes essential.

## Step 4 — Vendors see their own reports (scoped)

A vendor should view *their* payout reports — and that's your isolation model again (Chapter 28): a `GET /vendor/payouts` endpoint, scoped to `req.user`'s store, returns only that vendor's reports. A vendor can never see another's earnings. The payout feature inherits the security you built; you don't add new rules, you reuse them.

## Step 5 — Do it on your project (hands-on)

**1. Migrate a `payout_reports` table** — `store_id` FK, `period`, `payout_cents`, `order_count`, `created_at`, with `UNIQUE (store_id, period)` for idempotency. New migration (Chapter 12).

**2. Build the batch job** — a scheduled job (Chapter 54) that runs the set-based aggregation (Step 2) for the period and inserts a report per vendor, idempotently.

**3. Build `GET /vendor/payouts`** — scoped to the caller's store (Chapter 28), returning their reports.

**4. Prove correctness, efficiency, and idempotency:**

```bash
# run the payout job for a period (on demand for testing)
# → "generated payout reports for N vendors in 1 query"

# verify per-vendor totals match their completed orders
psql "$DATABASE_URL" -c "SELECT store_id, payout_cents, order_count FROM payout_reports WHERE period='2026-W22';"

# a vendor sees ONLY their own payout (isolation)
curl -s -H "authorization: Bearer $V" localhost:3000/vendor/payouts | jq '.[].payout_cents'

# run the job AGAIN for the same period → no duplicate reports (idempotent)
psql "$DATABASE_URL" -c "SELECT count(*) FROM payout_reports WHERE store_id='<id>' AND period='2026-W22';"   # → 1
```

One aggregation query producing every vendor's payout, each vendor seeing only their own, and a re-run creating no duplicates — that's a correct, scalable, safe payout system.

> 💡 **Hint — payouts are a *reporting* boundary; reconcile, don't trust blindly.** A real payout system would cross-check against payment records before actually moving money, and keep an audit trail of every report generated. You're building the *reporting* half (what each vendor is owed); the *paying* half (a payment provider transfer) is beyond this course's scope, but design the report as the auditable record it would feed. Note where real payment integration would attach.

> **📖 Mandatory read — before Chapter 56.** Read about **SQL aggregation (`GROUP BY`, `SUM`)** if you want a refresher, and a short piece on **batch processing patterns** (search *"batch job processing best practices"*). *Required: efficient aggregation is what makes batch jobs scale.*

> **Interesting to read.** Marketplace payouts are a genuinely hard domain — fees, refunds, chargebacks, holds, multi-currency, tax — which is why companies like Stripe built entire products (Connect) just for marketplace payments. Search *"stripe connect marketplace payouts"* to see the real-world complexity beyond the clean reporting you built.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 56:

- [ ] A **scheduled batch job** computes per-vendor payouts for a period from **completed** orders' snapshotted totals
- [ ] It uses a **single set-based aggregation** (`GROUP BY store_id`), not a query per vendor
- [ ] Reports are **idempotent** via a `UNIQUE (store_id, period)` key — a re-run creates no duplicates (you verified)
- [ ] `GET /vendor/payouts` is **scoped** so a vendor sees only their own reports
- [ ] Per-vendor totals **match** their completed orders
- [ ] You can explain why earlier decisions (per-vendor split, snapshots) make this aggregation correct

> **✍️ Log it (mandatory).** In `learning-log/55-payout-reports.md` — **decision** first, then **topics**: **(Decision)** Why compute payouts with one set-based query instead of looping per vendor, and why is `(vendor, period)` the right idempotency key? **(Topics)** (1) Why is a payout report a simple aggregation *because* of the per-vendor split and snapshots? (2) What is batch processing and how does it avoid N+1 at job scale? (3) Why is idempotency especially critical for a payout job? (4) How does the vendor payout endpoint reuse your isolation model?

*All boxes ticked and the log written? **That completes async and batch processing.** Your app does the right work at the right time, off the request path. The final stretch hardens it for the real world and ships it.*

---

Next: the features are done — now make the app safe to expose to the internet, starting with limiting abuse. → **[Chapter 56 — Rate limiting](56-rate-limiting.md)**
