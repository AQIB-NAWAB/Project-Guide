# Chapter 58 — Readiness and graceful shutdown

In production, your app **restarts constantly** — every deploy (Chapter 60), every scale-up, every crash recovery stops the old process and starts a new one. The question this chapter answers is: *what happens to the requests and jobs that are in-flight at the moment it stops?* If the process is just killed, a customer's checkout dies mid-transaction, a half-sent email job is abandoned, database connections leak. If it shuts down **gracefully** — finishing what it started before exiting — nobody notices the deploy at all. This is the polish that separates a hobby project from a production service, and it ties back to the health check you built way back in Chapter 7.

## Where we're headed

By the end your API and worker both handle shutdown signals gracefully — stop accepting new work, finish in-flight work, close connections cleanly, then exit — and your readiness check reflects when the app is draining, so traffic stops being routed to it before it goes.

## Step 1 — The cost of a hard kill

When a deploy stops your app by sending it a `SIGTERM` (the standard "please stop" signal), a naive app does nothing special and gets force-killed a few seconds later. In that moment:

- **In-flight requests die.** The customer mid-checkout gets a dropped connection — and if it wasn't atomic, partial data (but you made checkout atomic in Chapter 47, so it rolls back — that protection matters *here*).
- **In-flight jobs are abandoned.** A worker processing an image or email is killed mid-task — relying on retries (Chapter 52) to recover, but better not to drop it at all.
- **Connections leak.** Database and Redis connections aren't closed cleanly.

A graceful shutdown turns this violent stop into an orderly wind-down.

## Step 2 — The graceful shutdown sequence

On receiving `SIGTERM` (or `SIGINT`), the app should:

1. **Stop accepting new work.** The API stops taking new HTTP connections; the worker stops pulling new jobs from the queue.
2. **Signal "not ready."** Flip the **readiness** check (Chapter 7) to *not ready*, so the load balancer stops routing new requests to this instance *before* it disappears.
3. **Finish in-flight work.** Let the requests already being handled complete, and let the worker finish the job it's currently processing (within a timeout).
4. **Close connections** — database pool, Redis, the queue — cleanly.
5. **Exit.**

```
SIGTERM → readiness = false (stop new traffic)
        → stop accepting new requests / jobs
        → drain: finish in-flight requests & current job (up to a timeout)
        → close DB / Redis / queue connections
        → process.exit(0)
```

There's a **timeout** on the drain (you can't wait forever for a stuck request), but within it, work completes instead of dying.

## Step 3 — Readiness vs liveness, revisited

This is where Chapter 7's distinction pays off. During shutdown, the instance is still *alive* (the process runs, finishing work) but not *ready* (it shouldn't get new traffic). That's exactly the **liveness vs readiness** split: you flip *readiness* to false so the load balancer drains traffic away, while the process stays alive long enough to finish what it has. Without a readiness signal, the load balancer would keep sending requests to an instance that's trying to shut down. The health check you built "for the deploy platform" in Chapter 7 is precisely what makes zero-downtime deploys possible now.

## Step 4 — The worker needs it too

Your app is two processes (Chapter 50), and **both** need graceful shutdown. The worker's version: on `SIGTERM`, stop pulling new jobs, finish the current one (BullMQ supports closing the worker so it completes the active job before stopping), then close connections and exit. An abruptly-killed worker mid-job relies on at-least-once redelivery (Chapter 52) — graceful shutdown means it usually doesn't have to.

## Step 5 — Do it on your project (hands-on)

**1. Handle signals in the API** — on `SIGTERM`/`SIGINT`: flip readiness to not-ready, stop the HTTP server from accepting new connections, wait for in-flight requests to finish (with a timeout), close DB/Redis, exit.

**2. Handle signals in the worker** — stop consuming, finish the active job, close connections, exit.

**3. Wire readiness** — your Chapter 7 readiness endpoint reports not-ready once shutdown begins.

**4. Prove graceful shutdown:**

```bash
# start the API, fire a slow-ish request, then send SIGTERM mid-request
curl -s localhost:3000/some-endpoint &        # in-flight request
kill -TERM <api pid>                            # ask it to stop

# observe (in logs): readiness flips to not-ready, the in-flight request COMPLETES,
# connections close, then the process exits 0 — the request was NOT dropped

# readiness during drain
curl -s localhost:3000/health                   # → reports not-ready while draining
```

Seeing the in-flight request finish before the process exits — rather than being cut off — is graceful shutdown working. Do the same for the worker: send `SIGTERM` while it processes a job and confirm the job completes before exit.

> 💡 **Hint — there's a deadline; don't wait forever.** A graceful shutdown has a **grace period** (the platform gives you N seconds after `SIGTERM` before it sends `SIGKILL`, which can't be caught). So your drain must finish *within* that window — set your drain timeout below the platform's grace period, and if work isn't done by then, exit anyway (relying on atomicity and retries to keep things safe). Graceful means "finish quickly if you can," not "block the deploy indefinitely."

> **📖 Mandatory read — before Chapter 59.** Read about **graceful shutdown** (`SIGTERM` handling, connection draining) and revisit **liveness vs readiness** (Chapter 7). Search *"nodejs graceful shutdown SIGTERM"*. *Required: zero-downtime deploys (Chapter 60) depend on this.*

> **Interesting to read.** Zero-downtime deploys work precisely because of this dance: the platform starts new instances, waits for them to report *ready*, routes traffic to them, then sends `SIGTERM` to the old ones, which drain gracefully. Search *"rolling deployment graceful shutdown readiness"* to see how readiness + graceful shutdown together make "deploy with no downtime" possible.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 59:

- [ ] The **API** handles `SIGTERM`/`SIGINT`: flips readiness to not-ready, stops accepting new requests, **finishes in-flight requests**, closes connections, exits
- [ ] The **worker** handles shutdown: stops pulling jobs, **finishes the active job**, closes connections, exits
- [ ] The **readiness** check (Chapter 7) reports **not-ready while draining**
- [ ] You **sent `SIGTERM` mid-request and saw it complete** rather than drop
- [ ] The drain has a **timeout** within the platform's grace period
- [ ] You can explain how readiness + graceful shutdown enable zero-downtime deploys

> **✍️ Log it (mandatory).** In `learning-log/58-readiness-and-shutdown.md` — **decision** first, then **topics**: **(Decision)** Why handle `SIGTERM` gracefully instead of letting the process be killed, and why flip *readiness* (not liveness) during shutdown? **(Topics)** (1) Walk the graceful shutdown sequence. (2) How does the Chapter 7 readiness check enable draining traffic before exit? (3) Why does the worker need graceful shutdown too, and what protects it if killed anyway? (4) Why must the drain respect a timeout / grace period?

*All boxes ticked and the log written? Then continue. The app starts and stops cleanly — one last hardening pass closes the remaining security gaps before you ship.*

---

Next: a final security sweep — headers, CORS, limits, and the last threat-model rows — before the app meets the internet. → **[Chapter 59 — Security hardening](59-security-hardening.md)**
