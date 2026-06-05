# Chapter 54 — Scheduled jobs

Every job so far has been triggered by something *happening* — a registration enqueues an email, an upload enqueues thumbnailing. But a lot of real work isn't triggered by an event at all; it runs on a **schedule**, with nobody pressing a button: clean up expired tokens nightly, delete orphaned uploads, expire abandoned carts, generate daily reports. This chapter adds **scheduled (recurring) jobs** — work that runs itself on a clock — and tidies up several loose ends you've been accumulating (expired verification tokens from Chapter 23, orphaned objects from Chapter 43, stale carts from Chapter 44).

It's a short, practical chapter that completes your async toolkit: you can now run work *in response to events* (Chapters 51–53) and *on a recurring timer*.

## Where we're headed

By the end you have at least one scheduled job running on a recurring schedule (e.g. nightly cleanup of expired tokens and orphaned uploads), built on your existing queue, idempotent and safe to run on every fleet instance.

## Step 1 — Event-triggered vs scheduled

Two kinds of background work, both on the same queue infrastructure:

- **Event-triggered** (Chapters 51–53) — enqueued *when something happens* (a request, an upload). The trigger is an action.
- **Scheduled / recurring** — enqueued *by the clock*, on a repeating interval (every night at 2am, every hour). The trigger is *time*. This is the classic "cron job."

BullMQ supports recurring jobs directly (repeatable jobs with a cron-like schedule), so you don't need a separate cron system — you define a repeatable job and the queue enqueues it on the schedule. The same worker processes it.

## Step 2 — The cleanup work you've accumulated

You've left several "needs periodic cleanup" notes across the course — scheduled jobs are where they get handled:

- **Expired verification tokens** (Chapter 23) — single-use, expiring; the expired ones should be purged.
- **Expired/revoked refresh tokens** (Chapter 22) — old rows that no longer matter.
- **Orphaned uploads** (Chapter 43) — objects in storage with no matching `images` row (uploads that never got recorded).
- **Abandoned carts** (Chapter 44) — carts untouched for weeks.

These are housekeeping: nobody triggers them, but left undone they accumulate cruft. A nightly cleanup job sweeps them.

## Step 3 — Scheduled jobs must be idempotent and safe to overlap

Two correctness concerns specific to scheduled work:

- **Idempotency (still).** A scheduled job can be retried or, in a multi-instance deployment, accidentally enqueued by more than one scheduler. Running the nightly cleanup twice should be harmless — deleting already-deleted rows is a no-op, which makes cleanup naturally idempotent (a good property to design for, per Chapter 52).
- **Don't double-schedule across instances.** If you run *three* worker processes (Chapter 60), you don't want three copies of the nightly job. BullMQ's repeatable jobs handle this (the schedule lives in Redis, shared), but it's a real consideration — the *schedule* is defined once, centrally, not per instance. Be aware of where you register the repeatable job so it isn't registered three times.

## Step 4 — Keep scheduled work efficient

A cleanup job might touch a lot of rows. Two habits:

- **Do the work in the database where you can** — a single `DELETE ... WHERE expires_at < now()` beats loading rows into the app and deleting them one by one (the N+1 lesson, Chapter 36, applies to jobs too).
- **Batch large deletes** — deleting millions of rows in one statement can lock a table (echoes of the migration lock warning, Chapter 12); delete in chunks if volume is high.

## Step 5 — Do it on your project (hands-on)

**1. Define a recurring cleanup job** — a BullMQ repeatable job on a schedule (e.g. nightly) that:
- deletes expired verification tokens (Chapter 23) and expired/revoked refresh tokens (Chapter 22),
- optionally reconciles orphaned uploads (Chapter 43) and abandoned carts (Chapter 44).

**2. Make it idempotent and DB-efficient** (Step 3–4) — set-based deletes, safe to run repeatedly.

**3. Register the schedule once** (not per worker instance).

**4. Prove it runs and works:**

```bash
# create some expired data (e.g. a verification token with a past expiry)
psql "$DATABASE_URL" -c "INSERT INTO verification_tokens (...) VALUES (..., now() - interval '2 days');"

# trigger the scheduled job (run it on-demand once, or shorten the schedule for testing)
# → worker logs "cleanup: removed N expired tokens"

# verify the expired rows are gone, valid ones remain
psql "$DATABASE_URL" -c "SELECT count(*) FROM verification_tokens WHERE expires_at < now();"   # → 0
```

Seeing the job run on its own schedule and sweep the expired data — without anyone triggering it — is scheduled work in action. Confirm running it again is a clean no-op (idempotent).

> 💡 **Hint — test scheduled jobs by running them on demand.** Don't wait until 2am to know your nightly job works. Structure the job's logic as a plain function the schedule *calls*, so you can also invoke it directly in a test or a dev command. The *schedule* triggers it in production; the *function* is what you unit-test. Separating "when it runs" from "what it does" makes scheduled work testable like any other code.

> **📖 Mandatory read — before Chapter 55.** Read BullMQ's docs on **repeatable / scheduled jobs** and a short piece on **cron and recurring background work** (search *"bullmq repeatable jobs"*). *Required: Chapter 55's payout reports run on a schedule built on this.*

> **Interesting to read.** The humble cron job runs an astonishing amount of the internet's housekeeping — backups, report generation, cache warming, cleanup — quietly, on schedules nobody thinks about until one fails. Search *"things that run on cron"* to appreciate the invisible scheduled work behind every system.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 55:

- [ ] A **recurring scheduled job** runs on a defined schedule via your queue (no separate cron system)
- [ ] It performs real **cleanup** (expired tokens, and/or orphaned uploads / abandoned carts)
- [ ] The job is **idempotent** — running it twice is a harmless no-op (you verified)
- [ ] It does **set-based** DB work, not row-by-row
- [ ] The schedule is **registered once**, not per worker instance
- [ ] The job's logic is callable **on demand** for testing

> **✍️ Log it (mandatory).** In `learning-log/54-scheduled-jobs.md` — **decision** first, then **topics**: **(Decision)** Why use the queue's repeatable jobs rather than a separate cron system, and where must the schedule be registered to avoid duplication across instances? **(Topics)** (1) What's the difference between event-triggered and scheduled jobs? (2) Which accumulated cleanup tasks did you handle, and why do they need scheduling? (3) Why must scheduled jobs be idempotent? (4) How do you make a scheduled job testable without waiting for its schedule?

*All boxes ticked and the log written? Then continue. You can run scheduled work — now use it for the most valuable recurring job in a marketplace: telling each vendor what they're owed.*

---

Next: build the recurring batch job that aggregates each vendor's sales into a payout report. → **[Chapter 55 — Payout reports](55-payout-reports.md)**
