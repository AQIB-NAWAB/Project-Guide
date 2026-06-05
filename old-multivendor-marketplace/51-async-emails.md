# Chapter 51 — Async emails

Back in Chapter 23 you stubbed the verification email with a `// TODO: send via queue (Ch51)` note, and in Chapter 47 you deliberately kept the checkout confirmation email *outside* the transaction. This is Chapter 51 — the moment those debts come due. With the queue and worker in place (Chapter 50), you'll move email sending off the request path entirely: the API enqueues an email job and returns instantly; the worker delivers it. Registration and checkout get faster and more reliable, and a slow or down email provider can no longer break either.

It's your first *real* job, so it also makes concrete the patterns every job after it follows — small payload, load-by-id in the worker, and an external call that's now safely isolated from the request.

## Where we're headed

By the end, the verification email and checkout confirmation are sent by the worker via enqueued jobs; the API no longer waits for email delivery; and a failing email provider no longer fails registration or checkout.

## Step 1 — Replace the stub with a job

In Chapter 23, registration generated a verification token and (in dev) *logged the link* synchronously. Now the producer side becomes a single enqueue:

```
// in registration (the producer) — was: log/send the email inline
await emailQueue.add('send-verification-email', { accountId })   // enqueue, return immediately
```

Registration returns the moment the job is *queued*. The worker handles delivery:

```
// in the worker — handler for 'send-verification-email'
// load the account + its current verification token by accountId, render the email, send it
```

Note the payload is just `accountId` (Chapter 50) — the worker loads everything else it needs. Do the same for the checkout confirmation: the checkout handler, *after* the transaction commits (Chapter 47), enqueues `send-order-confirmation` with the `checkoutId`; the worker sends it.

## Step 2 — Real delivery (now that it's safe to add)

In Chapter 23 the email transport was a dev stub because synchronous email was the wrong place to integrate a provider. Now that sending happens in the worker, you can wire a **real email provider** (an SMTP service or transactional-email API) behind an email-sending module — still configured via your config (Chapter 6), with the provider credentials as validated env vars. In development you can keep a stub or use a local mail-catcher (a dev SMTP inbox tool); in production, the real provider. The worker doesn't care which — same as your database and storage abstractions.

## Step 3 — Why this is more reliable, not just faster

Moving email async buys you more than speed:

- **Registration/checkout no longer depend on the email provider's uptime.** If the provider is slow or down, the *account is still created* and the *order still placed* — the email job simply waits in the queue and is delivered when the provider recovers (with retries, Chapter 52). Previously, a provider outage meant users couldn't register.
- **Transient failures become retryable.** A request that fails can't easily retry the email; a *job* can be retried automatically (next chapter). The queue turns "fire and hope" into "fire and guarantee, eventually."

This is the resilience half of async (Chapter 49): the request path is decoupled from a flaky external dependency.

## Step 4 — Do it on your project (hands-on)

**1. Create an email queue** (or reuse your queue with named jobs) and worker handlers for `send-verification-email` and `send-order-confirmation`.

**2. Replace the Chapter 23 inline email** with `emailQueue.add('send-verification-email', { accountId })` in registration.

**3. Enqueue the checkout confirmation** after the checkout transaction commits (Chapter 47), with the `checkoutId`.

**4. Wire an email module** (real provider via config, or a local mail-catcher in dev).

**5. Prove it's async and resilient:**

```bash
# register → returns FAST; the email sends in the worker a moment later
time curl -s -X POST localhost:3000/auth/register -d '{"email":"new@x.com","password":"a-long-passphrase","role":"vendor"}' -H 'content-type: application/json'
# → returns in ~ms; watch the worker terminal log "sent verification email"

# resilience: stop your dev mail-catcher (simulate provider down), then register again
# → registration STILL succeeds (account created); the email job waits/retries instead of failing the request
```

Registration returning before the email sends, and *still succeeding* when the mail service is down, is the chapter's payoff — the Chapter 23 TODO closed properly.

> 💡 **Hint — never block the request on the job's *result*.** Enqueue and move on; don't `await` the email being *sent*. If a feature genuinely needs to know an email was delivered, that's a *status* the client polls or a webhook the provider calls — not the request waiting. The whole point is the request finishing before the work does. (And keep secrets out of job payloads — pass the `accountId`, let the worker fetch what it needs.)

> **📖 Mandatory read — before Chapter 52.** Read your email provider's **API/SMTP** docs and a short piece on **transactional email** (search *"transactional email best practices"*). *Required: Chapter 52 adds retries, which matter most for flaky external calls like email.*

> **Interesting to read.** "Send the email in a background job" is so standard that frameworks ship it by default — because the alternative (blocking signup on an email server) is a classic cause of slow, fragile onboarding. Search *"why send emails in background jobs"* to see the consensus you just implemented.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 52:

- [ ] The **verification email** (Chapter 23 TODO) is sent by the **worker** via an enqueued job; registration returns immediately
- [ ] The **checkout confirmation** is enqueued **after the transaction commits** (Chapter 47) and sent by the worker
- [ ] Email jobs carry a **small payload** (an id); the worker loads what it needs
- [ ] A **down/slow email provider no longer fails** registration or checkout (you tested it)
- [ ] A real provider (or local mail-catcher) is wired via **config**, not hardcoded
- [ ] The request never waits on the email's delivery result

> **✍️ Log it (mandatory).** In `learning-log/51-async-emails.md` — **decision** first, then **topics**: **(Decision)** Why was email the wrong thing to do synchronously in registration, and what does moving it to a job fix beyond speed? Why enqueue the confirmation email *after* the checkout commits, not inside the transaction? **(Topics)** (1) What does the producer side look like now versus the Chapter 23 stub? (2) Why does the job payload carry only an id? (3) How does async email make registration resilient to a provider outage? (4) Why must you never block the request on the job's result?

*All boxes ticked and the log written? Then continue. Jobs run in the background — but background work *fails*: providers blip, networks drop. Next you make jobs retry, and make them safe to run more than once.*

---

Next: a background job that fails should retry — but a retried job can run twice, so it must be idempotent. → **[Chapter 52 — Retries and idempotency](52-retries-and-idempotency.md)**
