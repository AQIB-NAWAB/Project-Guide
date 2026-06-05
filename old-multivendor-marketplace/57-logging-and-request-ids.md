# Chapter 57 — Logging and request IDs

When your marketplace is live and something breaks at 3am — a checkout fails, a vendor reports a wrong total, an endpoint gets slow — you won't be watching. **Logs are all you have.** And the difference between logs that *help* and logs that *don't* is enormous: a wall of unstructured `console.log("error!")` strings is nearly useless under pressure, while structured, correlated logs let you reconstruct exactly what happened to one request among thousands. You've been deferring real logging since Chapter 7 (don't log health-check noise), Chapter 17 (log full errors server-side), and Chapter 19 (never log passwords). This chapter builds it properly.

## Where we're headed

By the end your app emits **structured (JSON) logs** at appropriate levels, every request carries a **request ID** that ties all its log lines (and its error response) together, and secrets/PII are **redacted** — so you can trace any single request end to end.

## Step 1 — Structured logs, not strings

The naive approach is `console.log("user " + id + " did thing")` — human-readable, machine-useless. You can't search it, filter it, or aggregate it. **Structured logging** emits each log as a **JSON object** with fields:

```
// ❌ unstructured
console.log("checkout failed for user 42")

// ✅ structured (e.g. via pino, your Chapter 3 logger)
log.error({ event: "checkout_failed", userId: 42, checkoutId: "...", err }, "checkout failed")
```

Now a log aggregator can filter `event = "checkout_failed"`, group by `userId`, or alert on a spike — because the data is *fields*, not prose. Use a fast structured logger (pino, from your stack choice in Chapter 3) rather than `console.log`. Logs become queryable data, not a scroll of text.

## Step 2 — Log levels

Not every log is equally important. **Levels** let you control the noise:

- **`error`** — something failed and needs attention (a `5xx`, a job that exhausted retries). Ties to the `4xx`/`5xx` distinction from Chapter 17: log `5xx` as errors (your bugs), not `4xx` (expected client mistakes).
- **`warn`** — something off but handled (a retry, a rate-limit hit).
- **`info`** — significant events (a request completed, a checkout succeeded).
- **`debug`** — detailed diagnostics, off in production.

Set the level via config (Chapter 6) — verbose in dev, `info`+ in production. And recall Chapter 7: don't log every health-check poll, or `info` drowns in noise.

## Step 3 — Request IDs: correlation is everything

Here's the feature that makes production debugging possible. In a busy app, hundreds of requests interleave, so their log lines are jumbled together. A **request ID** — a unique id assigned to each incoming request and attached to *every* log line emitted while handling it — lets you filter to *one* request's complete story:

```
{ "reqId": "a1b2", "level": "info",  "msg": "POST /checkout" }
{ "reqId": "a1b2", "level": "info",  "msg": "split into 2 orders" }
{ "reqId": "a1b2", "level": "error", "msg": "payment provider timeout" }
```

Filter by `reqId: "a1b2"` and you see exactly what happened to that one checkout, in order, amid thousands of others. Generate the id in a middleware at the start of each request (or accept an incoming `X-Request-Id` from your load balancer so the trace spans systems), attach it to a request-scoped logger, and — crucially — **include it in your error responses** (the Chapter 17 error contract): when a user reports "I got an error," the `reqId` in their error body points you straight to the logs.

## Step 4 — What you must NOT log

Logs are a notorious leak vector — they're often less protected than the database, yet developers pour secrets into them. **Redact**:

- **Passwords** (Chapter 19's hint, now enforced) — never log `req.body` on auth routes without redacting the password.
- **Tokens** — access tokens, refresh tokens, API keys, the `JWT_SECRET`.
- **PII** beyond what you need — full payment details, etc.

Configure your logger's **redaction** (pino supports redacting fields by path) so even an accidental `log.info({ body })` strips the sensitive fields. A secret in a log file is a leak just like a secret in Git (Chapter 6) — treat logs as something an attacker might read.

## Step 5 — Do it on your project (hands-on)

**1. Set up a structured logger** (pino) configured via config — level from env, JSON output.

**2. Add a request-ID middleware** — generate (or accept) a request id, attach a request-scoped logger, and put the id on the request so handlers and the error handler can use it.

**3. Include `reqId` in error responses** — extend your Chapter 17 error handler so every error body carries the request id, and log full `5xx` errors (with the id) server-side.

**4. Configure redaction** — passwords, tokens, secrets stripped from all logs.

**5. Prove correlation and redaction:**

```bash
# make a request that does several things (e.g. a checkout), then grep your logs by its reqId
curl -i -X POST localhost:3000/checkout -H "authorization: Bearer $C" -H "idempotency-key: $(uuidgen)"
# → response includes an X-Request-Id / reqId; the error body (on failure) includes it too
# grep logs for that id → the request's full story on consecutive lines

# register, then check the logs: the password must NOT appear anywhere
curl -s -X POST localhost:3000/auth/register -d '{"email":"x@y.com","password":"SECRET-PASS","role":"customer"}' -H 'content-type: application/json'
grep -i "SECRET-PASS" your-logs   # → no matches (redacted)
```

Tracing one request by its id, and confirming the password never reached the logs, are the two proofs. The Chapter 19 "never log the password" promise is now enforced structurally.

> 💡 **Hint — the request id should follow the work, including into jobs.** When a request enqueues a background job (Chapters 51–53), pass the request id into the job payload and log it in the worker too. Then a checkout's request logs *and* its confirmation-email job logs share one id — so you can trace work that started in a request but finished in a worker. Correlation that stops at the queue boundary leaves you blind exactly where async bugs hide.

> **📖 Mandatory read — before Chapter 58.** Read about **structured logging** and **request/correlation IDs** (search *"structured logging request id correlation"*) and your logger's **redaction** docs. *Required: observability is what makes the production app debuggable.*

> **Interesting to read.** Mature systems extend request IDs into full **distributed tracing** (OpenTelemetry), following a single user action across many services with one trace id — the request id you just built is the first step of that whole discipline. Search *"distributed tracing correlation id"* to see where it leads.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 58:

- [ ] The app emits **structured (JSON) logs** via a real logger, with **levels** controlled by config
- [ ] Every request has a **request ID** on **all** its log lines and in its **error response** (Chapter 17)
- [ ] You can **filter logs to a single request** by its id and see its full story (you did)
- [ ] **Passwords, tokens, and secrets are redacted** — you confirmed a password never appears in logs
- [ ] `5xx` errors are logged in full server-side; `4xx` and health-check noise are not over-logged
- [ ] The request id is **propagated into background jobs**

> **✍️ Log it (mandatory).** In `learning-log/57-logging-and-request-ids.md` — **decision** first, then **topics**: **(Decision)** Why structured logs over `console.log` strings, and why does every request need a correlation id? **(Topics)** (1) How does a request id let you debug one request among thousands? (2) How do log levels and the `4xx`/`5xx` distinction (Chapter 17) shape what you log and alert on? (3) What must never be logged, and how do you enforce it? (4) Why propagate the request id into jobs?

*All boxes ticked and the log written? Then continue. You can see what your app does — now make sure it starts and stops *cleanly*, so deploys and restarts don't drop work.*

---

Next: deploys restart your app constantly — make it shut down gracefully so no request or job is dropped mid-flight. → **[Chapter 58 — Readiness and graceful shutdown](58-readiness-and-shutdown.md)**
