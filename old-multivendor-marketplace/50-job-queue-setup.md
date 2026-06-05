# Chapter 50 — Job queue setup

You've mapped the slow work that should run in the background (Chapter 49). Now build the infrastructure that runs it: a **job queue** and a **worker process**. This is foundational plumbing — once it's in place, every async feature this week (emails, image processing, reports, cleanup) is just "enqueue a job here, handle it in the worker there." This chapter builds and proves that pipeline end to end with a trivial job, so the real jobs in Chapters 51–55 slot straight in.

It also introduces a structural change you've been building toward since Chapter 4: your app is no longer *one* process. It's now **two** — the API and the worker — which shapes how you run it locally and deploy it (Chapter 60).

## Where we're headed

By the end you have a BullMQ queue backed by Redis, a separate worker process that consumes jobs, a test job that proves the API enqueues and the worker processes it, and the API returning immediately without waiting for the work.

## Step 1 — The pieces

From Chapter 49's model, made concrete with your stack:

- **The queue** — a named BullMQ queue, backed by **Redis** (already running since Chapter 5, configured via Chapter 6). The queue durably holds pending jobs.
- **The producer** — code in your API that adds a job to the queue (`queue.add('send-email', { ... })`) and returns immediately.
- **The worker** — a **separate process** (its own entry point) that connects to the same queue and processes jobs as they arrive.

The queue lives in Redis, so the API and worker don't talk directly — they communicate *through* the queue. The API drops a job in; the worker picks it up. They can even run on different machines (and will, in production).

## Step 2 — Two processes, not one

This is the structural shift. Back in Chapter 4 you split `app.ts` (the configured app) from `server.ts` (what starts it). Now you add a **third entry point**: a `worker.ts` that starts the worker. Your project runs as **two processes from the same codebase**:

```
process 1:  the API     (server.ts)  — handles HTTP, enqueues jobs
process 2:  the worker  (worker.ts)  — processes jobs from the queue
   both share: your config, your DB/Redis clients, your code
```

Locally you'll run both (two terminals, or a process manager). In production they're deployed and scaled *independently* (Chapter 60) — you might run one API and three workers, or vice versa. Add a `worker` script alongside `dev` (Chapter 4): `npm run dev` for the API, `npm run worker` for the worker.

## Step 3 — Jobs carry data, not code

A job is a small **message** — a job name plus a JSON payload describing what to do — not a function. The producer enqueues `{ name: 'send-verification-email', data: { accountId, ... } }`; the worker has a handler registered for that name that does the work. Keep payloads **small and serializable** (ids, not whole objects) — the worker can load what it needs from the database by id. (Passing a giant object through the queue is the same waste as N+1's per-row fetches; pass a reference.)

```
// producer (in the API) — enqueue and move on
await queue.add('send-verification-email', { accountId })   // returns immediately

// worker (in worker.ts) — handle the named job
// 'send-verification-email' → load the account by id, send the email
```

## Step 4 — Do it on your project (hands-on)

**1. Install BullMQ** and create a queue connected to Redis (via your config's `REDIS_URL`, Chapter 6).

**2. Create the worker entry point** (`worker.ts`) and a `worker` npm script. Register a handler for a test job name that just logs "processed job X".

**3. Enqueue a test job from the API** — a temporary `POST /debug/ping-job` (dev-only) that enqueues the test job and returns `202 Accepted` immediately.

**4. Prove the pipeline end to end:**

```bash
# terminal 1: the API
npm run dev
# terminal 2: the worker
npm run worker

# enqueue a job — the API returns INSTANTLY, before the work happens
curl -i -X POST localhost:3000/debug/ping-job    # → 202 Accepted, immediately

# watch terminal 2: the worker logs "processed job ..." a moment later
```

The API responding `202` *before* the worker logs the job is the whole point — the work is queued, not done, and the request didn't wait. Check Redis to see the queue (`redis-cli KEYS 'bull:*'`).

**5. (Optional) Add a queue dashboard** — BullMQ has UIs (e.g. Bull Board) that show jobs, failures, and retries. Useful for the rest of Week 4; not required.

> 💡 **Hint — `202 Accepted` is the honest status for "queued, not done."** When an endpoint *enqueues* work rather than completing it, `202 Accepted` ("I've accepted this for processing") is more truthful than `200 OK` ("done"). It tells the client the work is underway, not finished — relevant for things like "your report is generating." Use it where the response represents acceptance, not completion.

> **📖 Mandatory read — before Chapter 51.** Read the **BullMQ** docs on **queues, workers, and jobs** (search *"bullmq queue worker"*). *Required: every async feature in Chapters 51–55 builds on this setup.*

> **Interesting to read.** The producer/worker split is one of the oldest, most durable patterns in computing — the same idea runs print spoolers, CI pipelines, and email systems. Search *"message queue producer consumer pattern"* to see how universal the architecture you just built is.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 51:

- [ ] A **BullMQ queue** backed by **Redis** (via config) exists
- [ ] A **separate worker process** (`worker.ts` + `npm run worker`) consumes jobs
- [ ] The API **enqueues a test job and returns immediately** (`202`), and the worker processes it a moment later (you saw both)
- [ ] Jobs carry **small, serializable payloads** (ids), not whole objects or code
- [ ] You can run the app as **two processes** (API + worker) locally
- [ ] You can explain why the API and worker communicate through the queue, not directly

> **✍️ Log it (mandatory).** In `learning-log/50-job-queue-setup.md` — **decision** first, then **topics**: **(Decision)** Why run the worker as a separate process rather than processing jobs inside the API? Why pass ids in job payloads instead of whole objects? **(Topics)** (1) Describe the roles of producer, queue, and worker in your setup. (2) Why do the API and worker communicate through Redis rather than directly? (3) When is `202 Accepted` the right status code? (4) How does running as two processes change how you'll deploy (Chapter 60)?

*All boxes ticked and the log written? Then continue. The pipeline works with a test job — now put a real one through it, starting with the email you stubbed seven chapters ago.*

---

Next: cash in the Chapter 23 TODO — move the verification (and checkout) emails onto the queue. → **[Chapter 51 — Async emails](51-async-emails.md)**
