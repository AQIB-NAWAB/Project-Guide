# Chapter 60 — Deploy

This is the chapter the whole course was building toward: putting your marketplace **on the real internet**, at a real URL, over HTTPS, with real users able to reach it. Everything you've built converges here — the validated config (Chapter 6) that lets the same code run anywhere, the migrations (12) that build the schema on demand, the two-process architecture (50) of API and worker, the health check (7) and graceful shutdown (58) that make deploys safe, and the hardening (59) that makes exposure safe. Deploying isn't bolting something new on; it's *running what you have* in production conditions.

This chapter brings in the operational pieces — containers for production, a host, a database and Redis, HTTPS, DNS — as the deploy needs them (per the guide's closing arc). The result is a live application you can send someone a link to.

## Where we're headed

By the end your API and worker run in production (containerized), backed by managed (or pinned) Postgres and Redis, with secrets in the environment, migrations run on deploy, served over **HTTPS** at a real domain, with the health check and graceful shutdown enabling safe restarts.

## Step 1 — Production config = same code, real values

The foundation is already laid. Because your app reads everything from validated environment config (Chapter 6), production is *not* a code change — it's a different set of environment values: a production `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `S3_*`, email provider creds, and CORS origins. You ship **no `.env` file** to production (Chapter 6); the platform injects real secrets as environment variables, and your fail-fast config validation (Chapter 6) means a missing production secret crashes the deploy *immediately and loudly*, not mysteriously later. This is the payoff of doing config right in week one.

## Step 2 — Containerize for production

You've run Postgres, Redis, and MinIO in Docker since Chapter 5; now containerize *your app* for production. Write a **production `Dockerfile`** — typically **multi-stage**: one stage installs dependencies and builds your TypeScript, a lean final stage copies only the built output and production dependencies, so the image is small and contains no build tooling or source. The *same image* runs as your API or your worker — they're the same codebase with different entry points (`server.ts` vs `worker.ts`, Chapter 50), so you run the image two ways:

```
image: your-marketplace
  → run `node server.js`  → the API process(es)
  → run `node worker.js`  → the worker process(es)
```

This is dev/prod parity (the Twelve-Factor principle from Chapter 5) at its fullest: the container that runs in production is as close as possible to what you ran locally.

## Step 3 — Where it runs: the pieces

A production deployment of this app needs:

- **A host** — a PaaS (Railway, Render, Fly.io, etc.) or a VPS. A PaaS handles much of the below for you; a VPS means you wire it up (SSH in, run containers, configure a reverse proxy). Either is fine; a PaaS is faster to a live URL.
- **Managed Postgres and Redis** — in production you point at a *managed* database and Redis (backups, scaling, patching handled for you) rather than containers you operate. Thanks to config, your app doesn't care — it's just different URLs (the payoff promised back in Chapter 5).
- **Object storage** — real S3/R2 instead of local MinIO (Chapter 43), again just config.
- **The two processes** — deploy the API and the worker **separately** so they scale independently (Chapter 50): maybe one API and one worker to start, more of each under load.

## Step 4 — Migrations on deploy

Your schema is built by migrations (Chapter 12), and production is no exception. Run `npm run migrate` as a **deploy step** — *before* the new app version starts serving — so the database schema is up to date when the new code runs. This is why migrations had to be repeatable and ordered: the deploy pipeline runs them automatically against the production database. Never hand-edit the production schema (Chapter 12's golden rule, now at the highest stakes).

## Step 5 — HTTPS, DNS, and the last threat-model row

The final security row from Chapter 59 — **interception (MITM)** — closes here with **HTTPS/TLS**, so all traffic (credentials, tokens) is encrypted in transit:

- **DNS** — point your domain at the host (an A/CNAME record), so `marketplace.example.com` resolves to your app.
- **TLS certificate** — get an HTTPS certificate (most PaaS platforms provision one automatically; on a VPS, use a reverse proxy with Let's Encrypt). Enforce HTTPS (redirect HTTP → HTTPS), which your HSTS header (Chapter 59) then reinforces.

With TLS live, **every row of your threat model is closed.**

## Step 6 — The health check earns its keep

Now the endpoint you built in Chapter 7 does its real job: the host (or load balancer) polls `/health` to know when a freshly-deployed instance is **ready** for traffic, and (with graceful shutdown, Chapter 58) drains traffic from old instances before stopping them. That's what makes a deploy **zero-downtime** — new instances come up, report ready, take traffic; old ones drain and exit. The readiness check and graceful shutdown you built "for later" are exactly what make shipping a new version invisible to users.

## Step 7 — Do it on your project (hands-on)

**1. Write a production multi-stage `Dockerfile`**; build the image.

**2. Provision the host + managed Postgres/Redis + object storage**, and set all secrets as **environment variables** (no `.env` in prod).

**3. Configure the deploy** to run **migrations** first, then start the **API** and **worker** processes from the image.

**4. Set up DNS + HTTPS** at your domain; enforce HTTPS.

**5. Verify the live app:**

```bash
# health check green over HTTPS
curl -s https://your-marketplace.example.com/health        # → 200, checks ok

# the catalogue works against production data
curl -s https://your-marketplace.example.com/products | jq '.items | length'

# a real flow end to end: register → verify → open store → create product → it appears in the catalogue
# (do this through your deployed dashboard, Chapter 34)
```

A green health check over HTTPS, a working catalogue, and a full register-to-publish flow on the live URL — **your marketplace is on the internet.** Run it through a security-header scanner (Chapter 59's interesting read) for a final grade.

> 💡 **Hint — deploy early, deploy often (and you should have).** The best time to first deploy is *early* — a "hello world" deploy in week one surfaces config, secret, and platform issues while the app is simple, instead of all at once at the end. If this is your first deploy, expect to discover environment gaps now; note them, and next project, deploy on day one and redeploy continuously. A deploy pipeline you run constantly is reliable; one you run once is a gamble.

> **📖 Mandatory read — before you ship.** Read your chosen platform's **deploy docs**, and the **Twelve-Factor App** (revisit it whole now — search *"12 factor app"*) — you've implemented most of its twelve factors across this course; seeing them named together is a satisfying capstone. *Required: it's the framework your whole production setup quietly followed.*

> **Interesting to read.** Look back at how much had to be true for a *safe* deploy: config in env, repeatable migrations, health checks, graceful shutdown, separate scalable processes, secrets management. Search *"what zero downtime deployment requires"* and recognize that you built every prerequisite — deployment is less a step than the *sum* of good practices.

## Definition of Done

Things you can **see or run** — and the gate to the closing chapter:

- [ ] The app is **containerized** (multi-stage `Dockerfile`); the **same image** runs as API and worker
- [ ] **API and worker** run in production, backed by **managed/real** Postgres, Redis, and object storage
- [ ] **All secrets** are environment variables; **no `.env`** ships to production; config validates at startup
- [ ] **Migrations run automatically** on deploy, before the new version serves
- [ ] The app is served over **HTTPS** at a **real domain**; HTTP redirects to HTTPS (last threat-model row closed)
- [ ] `/health` is **green over HTTPS**, and a **full user flow works** on the live URL
- [ ] Deploys are **safe** (health check + graceful shutdown enable draining)

> **✍️ Log it (mandatory).** In `learning-log/60-deploy.md` — **decision** first, then **topics**: **(Decision)** Why is deploying "the same code with different config" rather than a code change — what made that possible? Why run migrations as a deploy step rather than by hand? **(Topics)** (1) How does a multi-stage Dockerfile + two entry points run your two processes? (2) Why managed Postgres/Redis in prod, and why doesn't your code care? (3) How do the health check (Chapter 7) and graceful shutdown (Chapter 58) enable zero-downtime deploys? (4) Which threat-model row does HTTPS close, and how?

*All boxes ticked, the log written, and your marketplace live? Then there's one thing left — not to build, but to *prove*. Turn your project into something that opens doors.*

---

Next: it's deployed — now document it, write the case study, raise the bar, and get ready to defend every decision you made. → **[Closing — Ship, document, prove it](99-closing.md)**
