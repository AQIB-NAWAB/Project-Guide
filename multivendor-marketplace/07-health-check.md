# Chapter 7 — The health check endpoint

Your app boots from validated config (Chapter 6) and can reach Postgres and Redis (Chapter 5). It runs — but right now the only way to know it's alive is to look at your terminal. That's fine on your laptop and useless everywhere else. The moment this app lives on a server, *machines* need to ask it "are you okay?" — many times a minute, with no human watching. This chapter builds the endpoint that answers that question, and it turns out to be a sneakily deep little feature: done naively it will cheerfully lie about being healthy, and a lying health check is worse than none at all.

We'll build it step by step. First *why* a machine needs to poll your app, then the obvious endpoint and the quiet way it deceives you, then the version that actually tells the truth.

## Where we're headed

By the end you'll have a small, unauthenticated endpoint — call it `GET /health` — that a deploy platform or load balancer can poll to decide whether to send your app traffic. It will report not just "the process is running" but "the process can actually do its job — its database and cache are reachable" — and it will return the **right status code** so the machine polling it can act correctly.

## Step 1 — Why does anything need this endpoint?

Picture your app deployed (Chapter 60 makes this real). Something in front of it — a load balancer or your hosting platform — is responsible for sending users to it. That thing has a problem: *how does it know your app is ready for traffic?* It can't read your logs. It can't tell a booting app from a crashed one from a healthy one. So it does the only thing it can: it **polls a known URL on a fixed schedule** and reads the response.

That URL is your health check. The contract is dead simple — the platform asks, your app answers:

```
GET /health        →  200 OK     "send me traffic, I'm fine"
GET /health        →  503        "hold traffic, I'm not ready"
```

This single endpoint drives real decisions: whether a fresh deploy starts receiving users, whether a sick instance gets pulled out of rotation, whether the platform restarts your container. Get it right and bad instances quietly remove themselves. Get it wrong and the platform sends real customers to an app that can't serve them.

## Step 2 — The obvious version (and the lie hiding in it)

The fastest health check you can write is a route that returns `200 OK` with a cheerful body:

```
GET /health  →  200 OK
{ "status": "ok" }
```

That's it — always answers `200`, always says `ok`. Deploy it and the platform is happy. So what's wrong?

Here's the trap, concretely. Your app's process can be running *perfectly* while the thing it depends on is dead. Imagine Postgres falls over — the connection drops, the database is unreachable. Your Node process is still up, still answering HTTP, so this endpoint still returns `200 OK` "I'm fine." The platform believes it and keeps routing customers to you. Every one of them hits a request that tries to read the database and fails. **Your health check said "healthy" while every real request was failing.**

That's the lie: this endpoint only proves *the web server can respond*. It says nothing about whether the app can actually do its job. A health check that can't tell "running" from "working" gives the platform false confidence at the exact moment you need it to pull you out of rotation.

To fix it, we need to be precise about *what* we're checking — and that's a real distinction worth knowing by name.

## Step 3 — The concept: liveness vs readiness

There are two different questions a platform asks, and conflating them is what caused the lie above:

- **Liveness — "is the process alive?"** Is the app running at all, or has it crashed/hung? If liveness fails, the right reaction is to **restart** the app. A liveness check should be *shallow and cheap*: it must not depend on the database, because you don't want the platform killing and restarting a perfectly good app just because the database had a hiccup.
- **Readiness — "can the app serve a request right now?"** Is it not just alive but *able to do its job* — dependencies reachable, warmed up, ready? If readiness fails, the right reaction is **don't send traffic** (but don't necessarily restart — it may recover on its own). A readiness check *does* check dependencies, because that's the whole point of it.

The difference matters because the two failures demand opposite responses. A dead process should be restarted; a live process whose database blipped should be *waited for*, not killed. Mix them up — make liveness check the database — and a brief database wobble triggers a restart storm, where the platform kills healthy apps and makes the outage worse.

> 💡 **Hint — which one is `/health`?** For this course, the endpoint you build is a **readiness** check: it verifies the app can actually serve requests, not just that the process is breathing. That's the one this project needs.

## Step 4 — The right approach: a readiness check that proves it can work

So your `/health` endpoint should answer the *useful* question: can this app serve a real request? To answer truthfully, it has to actually exercise its critical dependencies — the same Postgres and Redis it connects to through `config` (Chapter 6). Not pretend. Check.

What "check" means is deliberately lightweight — you're not running a real query, just proving the connection is alive:

- **Postgres:** issue the cheapest possible query that proves the connection works — `SELECT 1`. If it returns, the database is reachable. If it throws or times out, it isn't.
- **Redis:** send a `PING`. A `PONG` back means the cache/queue is reachable.

Then the endpoint reports the combined result. When everything answers, it's ready:

```
GET /health  →  200 OK
{
  "status": "ok",
  "checks": { "database": "ok", "redis": "ok" }
}
```

And when a dependency is unreachable, it says so honestly — and crucially returns a **failure status code**, not `200`:

```
GET /health  →  503 Service Unavailable
{
  "status": "degraded",
  "checks": { "database": "down", "redis": "ok" }
}
```

That `503` is the entire point. A machine polling this reads the *status code*, not your nice JSON — `200` means "send traffic," anything else means "hold." By returning `503` when Postgres is down, your app removes *itself* from rotation automatically. No human, no pager, no guesswork: the sick instance stops receiving customers until it recovers. The detailed `checks` body is for the human who later opens the URL to see *which* dependency failed.

Here's the whole interaction you're building:

```
  ┌──────────────┐   GET /health    ┌─────────────┐  SELECT 1   ┌──────────┐
  │ Load balancer│ ───────────────▶ │   Your app  │ ──────────▶ │ Postgres │
  │  / platform  │                  │             │  PING       ├──────────┤
  │              │ ◀─────────────── │  aggregates │ ──────────▶ │  Redis   │
  └──────────────┘  200 ok / 503    └─────────────┘             └──────────┘
        │
        └─ 200 → keep in rotation      503 → pull out of rotation
```

## Step 5 — The details that make it correct (and don't make it a liability)

A readiness check touches your live dependencies on every poll, and the platform may poll it every few seconds. That power cuts both ways — a careless health check becomes its own problem. Four rules keep it honest and safe:

1. **Keep the checks cheap.** `SELECT 1` and `PING`, nothing heavier — this endpoint is hit constantly, and a slow health check drags down the very app it's meant to watch. Run them through the connection your app already uses (the `db` and `redis` clients from Chapter 6), so the check fails the same way real traffic would.
2. **Always put a timeout on each check.** If Postgres is hung (not down — *hung*), an un-timed check waits forever, and your health endpoint hangs with it — which the platform reads as a failure too, but worse, because it ties up resources. Give each check a short timeout (a second or two) and treat a timeout as "down."
3. **Return the right status code, every time.** `200` only when *everything* needed is healthy; `503 Service Unavailable` when any critical dependency is down. The machine acts on the code — get this wrong and everything downstream is wrong.
4. **Leave it unauthenticated, but don't leak internals.** The platform polling it has no credentials, so `/health` must be public — but that means *anyone* can hit it, so the body shows *what* failed, never connection strings or stack traces that tell a stranger *how*:

   ```
   // ❌ never — a public endpoint just handed an attacker your internals
   { "status": "down", "error": "ECONNREFUSED postgresql://market:pw@10.0.1.5:5432/..." }

   // ✅ enough for you to debug, nothing useful to a stranger
   { "status": "degraded", "checks": { "database": "down", "redis": "ok" } }
   ```

> 💡 **Hint — "critical" is a judgement call.** Should a *single* failed dependency flip the whole endpoint to `503`? It depends whether that dependency is required to serve traffic. If your app genuinely cannot work without Postgres, then Postgres being down *should* return `503`. But a non-critical dependency being down might warrant a `200` with a warning in the body, because the app can still serve customers. For this course, treat Postgres as critical; think about where Redis falls given how you use it.

> **📖 Mandatory read — before Chapter 8.** Read a short piece on **health checks: liveness vs readiness** (search: *"liveness vs readiness health check"*). *Required: the distinction between "alive" and "ready" underpins how everything you deploy later gets monitored.*

> **Interesting to read.** Plenty of real outages have been *caused* by health checks, not caught by them: a check that ran an expensive query under load, or one wired so a single slow dependency made every instance report unhealthy at once — so the platform pulled *all* of them out of rotation and took down a healthy service. Search *"health check caused outage"*. It's the clearest argument for rules 1 and 2 above: the watchdog must never become the threat.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 8:

- [ ] A `GET /health` endpoint exists and is reachable without authentication
- [ ] It actively checks **Postgres** (a trivial `SELECT 1`) and **Redis** (a `PING`) through your `config`-driven connections — it does not just return a hardcoded `ok`
- [ ] When all dependencies are reachable, it returns **`200`** with a body showing each check
- [ ] When a critical dependency is down, it returns **`503`** (test it: stop your Postgres container with `docker compose stop postgres`, hit `/health`, confirm you get `503` — then start it again and confirm `200`)
- [ ] Each dependency check has a **timeout**, so a hung dependency can't hang the endpoint
- [ ] The response body leaks no secrets, connection strings, or stack traces

> **✍️ Log it (mandatory).** In `learning-log/07-health-check.md`: (1) What is the difference between a **liveness** and a **readiness** check, and why would checking the database in a *liveness* probe be a mistake? (2) Why is a health check that always returns `200 OK` worse than useless — describe the exact scenario where it causes harm. (3) Why must the endpoint return `503` (not `200`) when Postgres is down — what does the machine polling it do with that code?

*All boxes ticked and the log written? Then continue. You now have an app that can honestly report its own health — the foundation every later deploy and monitoring decision rests on.*

---

Next: the app stands and reports its health — time to model the world it serves. We start with the people and the things they own. → **[Chapter 8 — Modelling vendors, stores, and customers](08-model-vendors-stores-customers.md)**
