# Chapter 5 — Your local environment with Docker

Your app boots, but right now it has nothing to talk to. It needs the two services you chose in Chapter 3 — **PostgreSQL** and **Redis** — running somewhere it can reach. This chapter gets them running on your machine in a way that's reproducible, disposable, and identical for every learner. We'll take it slowly: first *where* services can run, then *what Docker is*, then how to use it.

## First: where do services like this even run?

You've picked Postgres and Redis. Before running anything, it helps to know your options, because there's more than one:

- **Managed cloud services.** A provider runs the database for you — hosted Postgres like Supabase, Neon, or AWS RDS; hosted Redis like Upstash or ElastiCache. You get backups, scaling, and no maintenance, but it costs money, needs accounts, and is far more than you want while developing on a laptop.
- **Installed natively on your machine.** You install Postgres and Redis directly through your operating system's package manager and run them as background services.
- **Run locally in containers with Docker.** You run each service in an isolated, throwaway box on your own machine.

For **local development** you want something free, fast to reset, and the same for everyone following this course — so we'll run them **locally with Docker**. (In production, Chapter 60, you'll most likely point at a *managed* service instead — and because of the configuration you set up in Chapter 6, your application code won't care which one it's talking to. That's the payoff of doing config properly.)

## Couldn't you just install them directly?

You could — `brew install postgresql redis` or the `apt` equivalent — and it would work today. It's just not the best option, for three quiet reasons: you'd run whatever **version** your package manager happens to give you (which drifts away from your teammates' and your production server's versions, breeding bugs that reproduce on one machine but not another), the services install **globally** and clutter your system, and a dev database that gets into a weird state is **annoying to wipe and rebuild**. We want our environment *declared, versioned, and disposable* — which is exactly what Docker gives us. So let's understand it.

## What is Docker?

Docker solves one of the oldest problems in software: *"it works on my machine."* Code that runs fine for you breaks for a teammate because their machine has a different version of something, a missing library, a different OS. Docker makes that impossible by shipping the software **together with everything it needs to run**, in a sealed box that behaves the same everywhere.

A few core ideas — these are all you need, and they'll click with analogies:

- **Container.** A lightweight, isolated box that holds one piece of software plus its dependencies, running on your machine but walled off from it. Think of it like a shipping container: whatever's inside, it loads onto any ship the same way.
- **Image vs container.** An **image** is the read-only *blueprint* — "PostgreSQL 16, configured like so." A **container** is a *running instance* of an image. The relationship is like a **recipe and a cooked dish**, or a **class and an object**: one blueprint, as many running instances as you like, each identical to start. You'll *pull* ready-made `postgres` and `redis` images — you don't build them yourself.
- **Docker Engine.** The program running on your computer that actually builds, starts, and stops containers. ("Docker Desktop" is Engine plus a GUI for Mac/Windows.)
- **Registry (Docker Hub).** The online library of prebuilt images. When you ask for `postgres:16`, Docker downloads that image from the registry the first time, then caches it.
- **Ports.** A container is isolated, so a service inside it isn't reachable by default. You **map a port** out — connecting a port on your laptop to a port inside the container — so your app on the host can reach Postgres in its box.
- **Volumes.** A container is *disposable*: anything written inside it disappears when the container is removed. A **volume** is storage that lives *outside* the container, so data you care about (your database) survives even when the container is destroyed and recreated. Remember this one — it's the mistake everyone makes once.
- **Docker Compose.** Rather than starting each container by hand with long commands, you describe *all* your services — Postgres and Redis together — in a single `docker-compose.yml` file, and bring them up with one command. This is how you'll run your environment.

That's the whole mental model. Now let's use it.

## Step 1 — Install Docker (if you haven't)

Before anything else, you need Docker on your machine. Install **Docker Desktop** (Mac/Windows) or **Docker Engine** (Linux) from the official guide — search *"install Docker Desktop docs.docker.com"* and follow your platform's instructions. Then confirm it's working:

```
docker --version          # prints a version
docker compose version    # prints a version
```

If both print versions, you're ready. (If `docker compose` isn't found, your install is older — check the install guide; modern Docker ships Compose built in.)

## Step 2 — Get the services running (start simple)

Create a `docker-compose.yml` at your project root. Start with the *minimum* to get Postgres and Redis up — don't reach for the advanced options yet. You're declaring two services:

```yaml
# docker-compose.yml — the minimal version (we'll harden it in Step 3)
services:
  postgres:
    image: postgres                 # official Postgres image (version pinned in Step 3)
    environment:
      POSTGRES_USER: market         # the DB user…
      POSTGRES_PASSWORD: localdevpw # …its password — fine in the file for LOCAL dev only
      POSTGRES_DB: marketplace      # the database created on first run
    ports:
      - "5432:5432"                 # map container port → your laptop so the app can connect

  redis:
    image: redis                    # official Redis image
    ports:
      - "6379:6379"
```

(Keep the official Compose file reference open as you write — see the read below.) Bring them up and check they're alive — never assume:

```
docker compose up -d        # start both, in the background
docker compose ps           # both should show as running
```

If you see both services running, you've already got a working local environment. Now let's make it *robust*.

## Step 3 — Make it production-ish (the refinements)

With the basics working, layer on the three things that turn a toy setup into a trustworthy one:

1. **Pin the image versions.** Use `postgres:16` and `redis:7`, never `postgres:latest`. *Why:* `latest` silently changes whenever a new version ships, which destroys reproducibility — and you want Postgres's **major version to match what you'll run in production**, so they behave the same.
2. **Give Postgres a named volume.** Attach a volume to Postgres's data directory. *Why:* without it, removing the container deletes your **entire database**. With it, your data survives restarts. (Redis is a cache/queue here, so losing its data on restart is usually fine — a volume there is optional.)
3. **Add a healthcheck to Postgres** (optional but professional). A container being "started" is not the same as the database being *ready to accept connections*. A healthcheck lets tools wait for genuine readiness — saving you from a race where your app tries to connect a split second too early.

Put together, the production-ish version looks like this — read the `# ←` comments to see exactly what changed:

```yaml
# docker-compose.yml — with the refinements
services:
  postgres:
    image: postgres:16              # ← pinned: match your production major version
    environment:
      POSTGRES_USER: market
      POSTGRES_PASSWORD: localdevpw
      POSTGRES_DB: marketplace
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data   # ← named volume: data survives container removal
    healthcheck:                          # ← wait until the DB truly accepts connections
      test: ["CMD-SHELL", "pg_isready -U market"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7                  # ← pinned

    ports:
      - "6379:6379"

volumes:
  pgdata:                           # ← declare the named volume referenced above
```

The three changes from the minimal version — pinned tags, the `volumes:` entry (plus its top-level declaration), and the healthcheck — are the whole climb from *"it runs"* to *"I trust it."*

> 💡 **Hint — the volume trap, made concrete.** `docker compose down` stops and removes the *containers* but **keeps** the named volumes — your data is safe. `docker compose down -v` *also* removes the volumes — your database is wiped clean. So: plain `down` (or `stop`) to pause for the day and keep your data; `down -v` when you deliberately want a fresh start. This one line saves a lot of "where did my data go?"

## Step 4 — Connect your app and prove it

Your app will reach these by URL. Name two environment variables now (you'll wire them into real, validated config in Chapter 6):

```
DATABASE_URL=postgresql://<user>:<password>@localhost:5432/<dbname>
REDIS_URL=redis://localhost:6379
```

The host is `localhost` because you mapped the containers' ports out to your laptop. (Once your app itself runs inside Docker — a deployment concern for Chapter 60 — services address each other by *service name* on Docker's internal network instead. File that away; it's a classic point of confusion.)

Now prove each service is reachable:

```
psql "$DATABASE_URL" -c "SELECT 1;"     # → returns 1
redis-cli -u "$REDIS_URL" ping          # → PONG
```

Two green checks, and your foundation is real. And make sure `.env` is in your `.gitignore` (you added it in Chapter 4) — those connection strings carry a password and must never reach Git.

> 💡 **Hint — two gotchas you may hit.** If `docker compose up` complains a **port is already in use**, something (often a native install from a past project) is already on `5432`/`6379` — stop it, or map a different host port like `"5433:5432"` and update `DATABASE_URL`. If `psql` says **connection refused** the instant after `up`, the container is probably still starting — wait a moment and retry; this is exactly the race a healthcheck removes when you automate startup later.

Here's the whole picture of what you just stood up — your app on the host, reaching two containers Docker manages:

```
 Your machine (host)
 ┌──────────────────────────────────────────────────────┐
 │  Node app ──DATABASE_URL──▶ localhost:5432 ─┐         │
 │           ──REDIS_URL─────▶ localhost:6379 ─┤         │
 │                                             ▼         │
 │   ┌──────────────────── Docker ──────────────────┐   │
 │   │  [ postgres:16 ] ── pgdata volume (persists)  │   │
 │   │  [ redis:7 ]                                  │   │
 │   └───────────────────────────────────────────────┘  │
 └──────────────────────────────────────────────────────┘
```

> **📖 Mandatory read — alongside this chapter.** Read the official Docker **"What is a container?" / Get Started** overview, and the **Compose file reference** section on **volumes** (search: *"docker compose volumes"*). *Required: you're making real decisions about ports and volumes; see the persistence behaviour in the docs, don't just take it from me.*

> **Interesting to read.** What you've just done has a name — **dev/prod parity**, one of the *Twelve-Factor App* principles: keep development, staging, and production as alike as possible so bugs can't hide in the gaps between them. Search *"twelve factor dev prod parity"* — you're now living a principle big teams treat as gospel.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 6:

- [ ] Docker is installed; `docker --version` and `docker compose version` both work
- [ ] A `docker-compose.yml` defines a **Postgres** and a **Redis** service
- [ ] Both images are **pinned** (no `:latest`); Postgres matches the major version you'll use in production
- [ ] Postgres has a **named volume**, and you confirmed data **survives** a `docker compose down` then `up` (and you know `down -v` wipes it)
- [ ] `docker compose up -d` starts both; `docker compose ps` shows them running
- [ ] `SELECT 1;` returns via `DATABASE_URL`; `redis-cli ping` returns `PONG` via `REDIS_URL`
- [ ] `.env` is git-ignored; no connection string or password is committed

> **✍️ Log it (mandatory).** In `learning-log/05-local-environment-docker.md`: (1) In your own words, what is a container, and how is it different from an image? (2) Why run these services in Docker rather than installing them natively — name the failure it prevents. (3) What does a volume protect you from, and what's the difference between `docker compose down` and `down -v`?

*All boxes ticked and the log written? Then continue. If `SELECT 1` or `PONG` won't come back, that's today's work — nothing past here runs without these two services.*

---

Next: the app and its services exist — now connect them the production way, with validated configuration and no secrets in code. → **[Chapter 6 — Configuration & secrets from the environment](06-config-and-secrets.md)**
