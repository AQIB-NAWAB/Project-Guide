# Chapter 52 — Retries and idempotency

Background jobs fail. The email provider has a blip, a network call times out, a third-party API returns a transient error. In a request you'd just return an error and let the user retry; but a job has no user waiting — so if a job fails and nothing retries it, that verification email is simply *never sent*, silently. The whole point of moving work to a queue (Chapter 49) was reliability, and reliability means **automatic retries**. But retries introduce a twist you've met before: a job that retries might run **more than once** — so jobs must be **idempotent**, exactly as checkout had to be (Chapter 48).

This chapter makes your jobs resilient (they retry on failure) *and* safe (running twice does no harm) — the two properties that make a queue trustworthy.

## Where we're headed

By the end your jobs retry with exponential backoff up to a limit, permanently-failed jobs land somewhere visible (a dead-letter), and your jobs are idempotent so an at-least-once retry can't double-send or double-charge.

## Step 1 — Retries with backoff

When a job throws, BullMQ can retry it automatically — but *how* it retries matters:

- **Max attempts** — retry a few times (say 3–5), then give up. Infinite retries on a permanently-broken job just churn.
- **Exponential backoff** — wait *longer* between each attempt (1s, then 4s, then 16s…), not immediately. A retry one millisecond after a failure usually fails again (the provider is still down); backing off gives the transient problem time to clear, and avoids hammering a struggling dependency.

```
job fails → wait 1s → retry → fails → wait 4s → retry → fails → wait 16s → retry → give up after N
```

Configure attempts and backoff on the job (or queue). This turns a momentary provider outage into a non-event: the job quietly retries and succeeds once the provider recovers — the resilience you set up async email for (Chapter 51).

## Step 2 — At-least-once delivery means run-twice is possible

Here's the catch that makes idempotency mandatory. Queues like BullMQ give **at-least-once** delivery: they guarantee a job runs *at least* once, but under certain failures (a worker crashes *after* doing the work but *before* marking the job complete) a job can run **more than once**. Combined with retries, this means: **assume every job may execute two or more times.**

If your `send-verification-email` job runs twice, the user gets two emails — annoying but survivable. If a hypothetical `charge-payment` or `create-payout` job runs twice, you've double-charged or double-paid — a real incident. So the rule, identical to checkout (Chapter 48): **jobs must be idempotent** — running them twice has the same effect as running them once.

## Step 3 — Making a job idempotent

The technique is the one from Chapter 48, applied to workers: make the job's effect a no-op on repeat. Depending on the job:

- **Guard with state.** Before doing the work, check whether it's already been done. A `send-verification-email` job can check "is this token already used/sent?"; an image job can check "do thumbnails already exist for this key?"; a payout job records a `(vendor, period)` key and skips if it's already recorded.
- **Use a natural unique key.** Many jobs have an inherent idempotency key — the order id, the image key, the `(vendor, period)` pair — that a `UNIQUE` constraint or a check-before-act turns into a safe "already done? skip."

```
// idempotent job handler
if (already done for this key) return;   // no-op on repeat
do the work;
record that it's done (atomically where it matters);
```

The mental model transfers directly from checkout: **make repeat execution harmless.** Design every job assuming it *will* sometimes run twice.

## Step 4 — When retries run out: dead-letter

A job that fails all its attempts shouldn't vanish silently. Failed jobs should land somewhere visible — a **dead-letter queue** or a "failed" state you can inspect and alert on — so you *know* the verification email for account X never sent after 5 tries, rather than discovering it from a confused user. Failure handling is part of reliability: retry the transient, surface the permanent.

## Step 5 — Do it on your project (hands-on)

**1. Add retries + backoff** to your email jobs (and future jobs): a max-attempts and exponential-backoff config.

**2. Make the email job idempotent** — guard so a re-run doesn't send a duplicate (e.g. check the token hasn't already been consumed/sent, or record a sent-marker).

**3. Prove retry and idempotency:**

```bash
# RETRY: make the email transport fail the first 2 times (e.g. throw on attempts 1–2), then succeed
# enqueue the job → watch the worker: attempt 1 fails, backoff, attempt 2 fails, backoff, attempt 3 SUCCEEDS
# the email is delivered despite the transient failures — no user impact

# IDEMPOTENCY: force the same job to run twice (re-enqueue with the same key / simulate double delivery)
# → the second run is a NO-OP; exactly one email sent, not two
```

Seeing a job survive transient failures via backoff, and seeing a doubled job send only one email, are the two proofs. **4.** Confirm a job that exhausts all attempts lands in the **failed/dead-letter** state where you can see it.

> 💡 **Hint — distinguish *transient* from *permanent* failures.** Retrying a transient error (provider timeout) is right; retrying a *permanent* one (the email address is malformed, the account was deleted) just wastes attempts and delays the dead-letter. Where you can, detect permanent failures and fail *fast* (don't retry), while letting transient ones back off and retry. Not every error deserves five attempts.

> **📖 Mandatory read — before Chapter 53.** Read BullMQ's docs on **retries, backoff, and failed jobs**, and revisit **Stripe's idempotency guide** (Chapter 48) with workers in mind. *Required: Chapter 53's image job must be idempotent and retryable too.*

> **Interesting to read.** "At-least-once delivery" is a fundamental property of distributed message systems — exactly-once is famously hard/impossible in general — which is *why* idempotent consumers are the standard solution everywhere from Kafka to SQS to BullMQ. Search *"at least once delivery idempotent consumer"* to see why "make the work safe to repeat" is the universal answer.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 53:

- [ ] Jobs **retry with exponential backoff** up to a max attempts; you saw a transient failure recover via retries
- [ ] Jobs are **idempotent** — you ran one twice and it had the **effect of running once** (no duplicate email)
- [ ] A job that **exhausts retries** lands in a visible **failed/dead-letter** state
- [ ] You can explain **at-least-once** delivery and why it forces idempotency
- [ ] (Where possible) **permanent** failures fail fast rather than retrying pointlessly

> **✍️ Log it (mandatory).** In `learning-log/52-retries-and-idempotency.md` — **decision** first, then **topics**: **(Decision)** Why must background jobs retry, and why does that *force* them to be idempotent? Why exponential backoff rather than immediate retry? **(Topics)** (1) What is at-least-once delivery and how can a job run twice? (2) How did you make your email job idempotent? (3) Why should a job that exhausts retries go to a dead-letter rather than vanish? (4) Why distinguish transient from permanent failures? (Tie idempotency back to Chapter 48.)

*All boxes ticked and the log written? Then continue. Your jobs are reliable and safe to repeat — apply them to the heavier task you deferred back at uploads: turning raw photos into thumbnails.*

---

Next: put the reliable queue to work on real CPU-heavy work — generate image thumbnails in the background. → **[Chapter 53 — Async image processing](53-async-image-processing.md)**
