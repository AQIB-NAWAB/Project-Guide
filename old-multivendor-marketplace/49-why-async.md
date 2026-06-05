# Chapter 49 — Why async?

Across the last three weeks you've quietly accumulated a list of work that doesn't belong in a request. The verification email you *stubbed* in Chapter 23 with a `// TODO: queue` note. The image thumbnails you deferred in Chapter 43. The checkout confirmation email you pushed *outside* the transaction in Chapter 47. The payout reports you'll need (Chapter 55). All of it shares a shape: it's **slow**, it can **fail and need retrying**, and the user doesn't need to *wait* for it. Doing that work inside the request is what makes apps feel sluggish and fragile. Week 4 fixes it.

This concept chapter explains *why* and *how* to move slow work off the request path — and draws a firm line around the *kind* of async this course uses, because "async" is a word that hides several very different architectures.

## What you'll be able to explain

By the end you can explain the cost of doing slow work synchronously, the producer–queue–worker model, what belongs async and what doesn't, and why this course uses a **job queue** specifically (not event-driven/pub-sub/CQRS).

## Section 1 — The cost of doing slow work in the request

When your registration handler sends an email *inline*, the user's request can't finish until the email provider responds. That's bad in three concrete ways:

- **The user waits** for work they don't care about the timing of. Registration should feel instant; instead it's gated on an email server hundreds of milliseconds (or seconds) away.
- **It ties up your server.** A worker process blocked waiting on an email API can't serve other requests. Slow external calls inside requests throttle your whole app.
- **It couples your uptime to theirs.** If the email provider is down, your *registration* fails — even though the account could have been created fine. One slow dependency takes down an unrelated feature.

The same logic applies to image processing, report generation, and checkout confirmations. None of it should block the user.

## Section 2 — The model: producer → queue → worker

The solution is to **do the slow work later, somewhere else**. Instead of performing the task in the request, you *enqueue a job* describing it, and a separate **worker** process performs it in the background:

```
API request (producer)  →  "send verification email to X"  →  [ QUEUE ]  →  worker picks it up → sends it
        │                                                                         (later, separately)
        └─ returns to the user IMMEDIATELY (job is queued, not done)
```

Three roles:

- **Producer** — your API, which *enqueues* a job and returns to the user right away.
- **Queue** — a durable list of pending jobs (you'll use **Redis** via **BullMQ** — your Redis is already running from Chapter 5, doing double duty as cache and queue).
- **Worker** — a *separate process* that consumes jobs and does the actual work, at its own pace.

The request returns in milliseconds; the email sends a moment later in the worker. The user never waits, your API stays responsive, and a slow email provider can't break registration.

## Section 3 — What belongs async (and what doesn't)

Not everything should be a job. The test:

- **Move it async** when it's **slow**, **retryable**, and the user **doesn't need the result in the response**: sending email, processing images, generating reports, syncing to external systems.
- **Keep it sync** when the user **needs the result now**: anything whose output is part of the response (the catalogue, creating an order and showing it, validating input). You can't defer work the user is literally waiting to see.

Your accumulated list — verification email, checkout confirmation, thumbnails, payout reports — is all firmly in the "async" column. The checkout *itself* stays synchronous (the customer needs their orders confirmed); only the *email about it* goes async (Chapter 47's seam).

## Section 4 — The line this course draws: job queue, not events

"Async" spans several architectures, and they are *not* interchangeable. This course uses **job queues** (BullMQ) — and deliberately **not** event-driven architecture, pub/sub, or CQRS. The distinction matters:

- **Job queue (this course):** "do this specific task later" — a producer enqueues a concrete unit of work for a worker to execute. Direct, simple, easy to reason about and retry.
- **Event-driven / pub-sub (out of scope):** "this *happened*; whoever cares can react" — publishers emit events, many independent subscribers respond. Powerful for decoupling many services, but a different mental model with its own large set of concerns (event schemas, ordering, eventual consistency).
- **CQRS / event sourcing (out of scope):** separating reads from writes and storing state as a log of events. A heavyweight pattern for specific problems.

For a single marketplace application moving slow tasks off the request path, a **job queue is exactly the right tool** — and reaching for event-driven architecture here would be over-engineering. Knowing *which* async you mean is itself an engineering skill; this course picks the simplest one that fits and stays with it.

## Section 5 — Map your async work (hands-on)

**1. List the async jobs your app needs**, in `docs/ASYNC.md`, with which chapter implements each:
- **Verification email** (the Chapter 23 stub) → Chapter 51
- **Checkout confirmation email** (the Chapter 47 seam) → Chapter 51
- **Image thumbnail processing** (the Chapter 43 defer) → Chapter 53
- **Payout report generation** (scheduled/batch) → Chapter 55
- **Cleanup of expired tokens / orphaned uploads** (scheduled) → Chapter 54

**2. For each, note** why it qualifies (slow / retryable / result-not-needed-in-response) and what its job payload would carry.

**3. Mark the producers.** Find the `// TODO: queue` seams you left (Chapters 23, 43, 47) — these are exactly where you'll enqueue jobs starting in Chapter 50. Seeing them as a list makes Week 4 a matter of *connecting* work you already deferred.

> **📖 Mandatory read — before Chapter 50.** Read an intro to **background job queues** and skim the **BullMQ** overview (search *"bullmq getting started"*). Optionally read a short "job queue vs event-driven / pub-sub" comparison to cement the Section 4 distinction. *Required: Chapter 50 sets up the queue and worker.*

> **Interesting to read.** Almost every app you use runs a fleet of background workers — sending your emails, resizing your uploads, generating your reports — entirely invisibly, because the whole point is that *you* never wait for them. Search *"background job processing architecture"* to see how central this quiet infrastructure is.

## Key takeaways

You can now explain:

- Doing **slow work in a request** makes users wait, ties up servers, and couples your uptime to slow dependencies.
- The **producer → queue → worker** model moves that work to a separate process so the request returns immediately.
- Work goes **async** when it's slow, retryable, and not needed in the response; it stays **sync** when the user needs the result now.
- This course uses a **job queue** (BullMQ), *not* event-driven/pub-sub/CQRS — the right-sized tool for moving tasks off the request path, and choosing it is itself a judgement.

> **✍️ Log it (mandatory) — this is the gate.** In `learning-log/49-why-async.md` — **decision** first, then **topics**: **(Decision)** For *your* app, why does the verification email belong async but creating the order stay sync? Why a job queue rather than an event-driven architecture for this project? **(Topics)** (1) Name the three costs of doing slow work inside a request. (2) Describe the producer–queue–worker model. (3) Give the test for whether work should be async, applied to two examples from your app. (4) How does a job queue differ from pub/sub, and why is it the right choice here?

*Answered all of it, and committed `ASYNC.md`? Then continue. You know what to move and why — now build the queue and worker that will run it.*

---

Next: stand up the job queue and a worker process — the infrastructure every async feature this week runs on. → **[Chapter 50 — Job queue setup](50-job-queue-setup.md)**
