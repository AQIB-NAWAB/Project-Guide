# Chapter 4 — Project bootstrap & structure

You've decided *what* you're building with. Now you create the thing those tools live in: the application skeleton. We do this **before** standing up databases (next chapter), because the app doesn't need a database to *exist* — and it's far easier to bring the services online once there's an app waiting to talk to them.

This chapter is foundational rather than flashy — nothing a user sees gets built here. But the one real decision inside it, **how you lay out your code**, is one you'll either thank or curse yourself for across the next 56 chapters. So we'll take it slowly and in detail, with the concrete commands and config you'll actually use.

## The point of this chapter

A **TypeScript project that starts with a single command**, configured strictly, and organised so it won't collapse as the marketplace grows from one feature to dozens. By the end, one script boots the app and keeps it alive, and your folders are ready for everything Weeks 1–4 will add.

## Step 1 — Initialise the project

Start an empty Node project and add TypeScript plus a dev runner. The package manager is your call (npm, pnpm, or yarn); the steps are the same:

```bash
# inside an empty project folder
npm init -y                                   # creates package.json
npm install --save-dev typescript tsx @types/node
npx tsc --init                                # creates tsconfig.json
```

What each piece is, so nothing is magic:

- **typescript** — the compiler (`tsc`) that type-checks your code and turns it into plain JavaScript.
- **tsx** — runs TypeScript files *directly*, with hot reload, so during development you don't have to compile-then-run on every change.
- **@types/node** — the type definitions for Node's built-ins, so TypeScript understands `process`, `path`, `Buffer`, and friends.

That's the toolchain. Now the decision that actually matters.

## Step 2 — Choose your folder structure (the real decision)

Both tempting shortcuts disappoint:

- **No structure** — everything in one file, or a flat `src/` where routes, business logic, and database calls pile together. Fast for a day; unfindable by the time you have vendors, products, orders, auth, carts, and jobs.
- **A random "ultimate starter" off the internet** — you inherit fifteen folders you didn't design and now maintain decisions you never made (and can't explain — see Chapter 3).

Choose deliberately instead, on a known axis. There are two mainstream philosophies.

**Layer-based** — group by technical role. Everything of one *kind* lives together:

```
src/
  controllers/    # every HTTP handler, for every domain
  services/       # every piece of business logic, for every domain
  repositories/   # every database access, for every domain
```

**Feature- / module-based** — group by domain. Everything about one *thing* lives together:

```
src/
  modules/
    vendors/
      vendors.routes.ts     # HTTP endpoints for vendors
      vendors.service.ts    # business logic
      vendors.repo.ts       # database access
      vendors.schema.ts     # request/response validation (Module 3)
    products/
      products.routes.ts
      products.service.ts
      products.repo.ts
      products.schema.ts
    orders/
      ...
  shared/
    config.ts               # validated env config (Chapter 6)
    db.ts                   # the Postgres client / ORM instance
    redis.ts                # the Redis client
    errors.ts               # shared error types (Module 3)
    logger.ts               # the logger (Module 14)
  app.ts                    # builds & wires the framework instance (does NOT listen)
  server.ts                 # starts the app listening — the entry point
```

The trade-off is **cohesion vs familiarity**. Layer-based is what most tutorials show, so it *feels* familiar — but watch what happens when you add a feature. Picture shipping "discount codes" in Week 5:

- **Layer-based:** you add a file to `controllers/`, another to `services/`, another to `repositories/` — three folders that already hold a dozen unrelated things — and you touch all three to ship one feature. Those folders grow without bound, and related code is scattered.
- **Feature-based:** you add one folder, `modules/discounts/`, with everything about discounts inside it. Nothing else moves.

That's **high cohesion** (things that change together live together) and **low coupling** (modules don't reach into each other's guts). Here's the honest comparison of the two, so you're choosing with eyes open:

| | Layer-based | Feature / module-based |
|---|---|---|
| **Pros** | Familiar — most tutorials use it; dead simple for a tiny app; "all the controllers" sit in one place | High cohesion — everything about one domain in one folder; features are self-contained; the app scales by *adding* folders; a whole feature is easy to find, move, or delete |
| **Cons** | Folders grow without bound; one feature is scattered across three places; related code drifts apart as the app grows; merge conflicts cluster in the same shared folders | Slightly less familiar at first; a touch more upfront structure than a trivial app needs |
| **Best for** | Small apps with a handful of endpoints | Apps with many distinct domains — **exactly a marketplace** |

### Your choice for this course (mandatory)

**Use feature / module-based — it's required, not a preference.** This is one of the few decisions the course *makes for you*, on purpose: the rest of the chapters are written assuming it. Later you'll read instructions like *"add this endpoint in `modules/products/products.routes.ts`"* or *"scope the query in `orders/orders.repo.ts`"* — and if you built layer-based instead, none of those references will line up and you'll be mentally translating every chapter. A foundational choice like this has to be *settled* so everything after it can build on solid ground.

So: pick feature/module-based now, adapt the exact file names to your framework (if you chose NestJS, its modules already are feature-based; with Fastify/Express you create the folders yourself), and commit to it. Within that structure the names are yours — the only test is *a new reader should be able to guess where product code lives without asking.*

## Step 3 — Configure TypeScript to actually help you

Open the `tsconfig.json` you generated and set the options that carry real weight. Here are the ones to care about, annotated:

```jsonc
// tsconfig.json — the options that matter most
{
  "compilerOptions": {
    "strict": true,             // every safety check ON — the whole reason you chose TypeScript
    "target": "ES2022",         // the JS version to compile down to (modern Node runs this fine)
    "module": "NodeNext",       // use Node's modern module system
    "moduleResolution": "NodeNext",
    "rootDir": "src",           // your TypeScript source lives here
    "outDir": "dist",           // compiled JavaScript is emitted here (git-ignored)
    "esModuleInterop": true,    // smooth interop with older CommonJS packages
    "skipLibCheck": true,       // don't re-type-check dependencies' .d.ts files (faster builds)
    "sourceMap": true           // map runtime errors back to your .ts line numbers
  },
  "include": ["src/**/*"]
}
```

The single most important line is **`"strict": true`**. Strict mode is what forces you to handle the possibly-`undefined` vendor, the maybe-null lookup, the wrong-shaped argument — *at compile time, in front of you*, instead of at runtime, in front of a customer. Turning it off to "move faster" silently throws away the exact safety you adopted TypeScript to get. Leave it on from line one.

Note also `rootDir: src` and `outDir: dist`: your TypeScript lives in `src/`, and compiling produces plain JavaScript in `dist/`. That split is why **development runs your `.ts` on the fly (via tsx)** while **production runs the compiled `.js` in `dist/`** — a distinction worth holding onto.

## Step 4 — Wire up your npm scripts

Add the handful of scripts you'll run all course, so nobody (including future-you) has to remember the incantations:

```jsonc
// package.json (scripts section)
{
  "scripts": {
    "dev":       "tsx watch src/server.ts",   // hot-reload dev server — your everyday command
    "build":     "tsc",                        // type-check and compile src/ → dist/
    "start":     "node dist/server.js",        // run the COMPILED app — this is what production runs
    "typecheck": "tsc --noEmit"                // check types without emitting (handy in CI later)
  }
}
```

Read the difference between `dev` and `start` carefully, because it's the dev-vs-prod model in two lines: **`dev` runs your TypeScript directly with reload for fast feedback; `start` runs the already-compiled JavaScript, which is what you deploy.** Same app, two ways to run it.

## Step 5 — A skeleton that boots

Now the smallest possible running app. Notice the structure splits the entry into two files, and the split is deliberate:

- **`src/app.ts`** *builds and wires* the framework instance — registers middleware and your module routes — and **returns** it. It does **not** start listening.
- **`src/server.ts`** imports that app and **starts it listening** on a port. It's the only file that actually "runs the server."

Why bother splitting them? **Testability.** When you test data isolation in Chapter 30, you'll want to import the fully-wired app and fire fake requests at it *without* opening a real network port — which is only possible if "build the app" and "start listening" are separate steps. Setting it up now costs nothing and saves a refactor later.

Your goal for this chapter is just to run `npm run dev` and watch the app come up and stay up — maybe with a single placeholder response. **Don't** add the real `/health` endpoint or read the port from configuration yet; reading config from the environment is Chapter 6, and the health check is Chapter 7. Resist the urge to run ahead — those chapters teach those properly.

## Step 6 — Lock in the daily habits

Three things before you call this done — the working rhythm from the intro, starting for real:

- **`git init`** and make your **first commit.** From here, you commit every day.
- Add a **`.gitignore`** so the things that must never be committed never are:

```gitignore
# .gitignore
node_modules/      # dependencies — reinstalled from package.json, never committed
dist/              # build output — regenerated by `npm run build`
.env               # secrets — never, ever in Git
.env.*.local
```

- Optionally add **ESLint + Prettier.** Not required, but consistent formatting keeps 60 chapters of code readable, and a linter catches a class of mistakes before you even run.

```bash
git init
git add -A
git commit -m "chore: bootstrap TypeScript project skeleton"
```

> **📖 Mandatory read — before you lock in a structure.** Read one short piece comparing **layer-based vs feature-based (modular) project structure** (search: *"feature based vs layered folder structure node"*), and skim your framework's recommended project layout. *Required: you're choosing the shape of the entire codebase; make it an informed decision, not a copied one.*

> **Make it yours.** A clean early build-in-public note: *"How I structured my marketplace backend — and the trade-off I chose."* Short, opinionated, with your folder sketch and the "adding a feature" comparison above.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 5:

- [ ] `npm run dev` starts the app and it stays listening on a port
- [ ] `tsconfig.json` has **`"strict": true`**, with `rootDir: src` and `outDir: dist`
- [ ] `dev`, `build`, and `start` scripts exist in `package.json` and you understand how `dev` and `start` differ
- [ ] `npm run build` compiles `src/` to `dist/`, and `npm start` runs the compiled output
- [ ] The project uses **feature/module-based** structure (**required** for this course), with `app.ts` (wiring) split from `server.ts` (listening)
- [ ] `git init` done, a **first commit** made, and `.gitignore` excludes `node_modules/`, `dist/`, and `.env`
- [ ] No secrets committed (there shouldn't be any yet — keep it that way)

> **✍️ Log it (mandatory).** In `learning-log/04-project-bootstrap.md`: (1) Which folder structure did you choose, and *why* — walk through "adding a feature" in both styles to justify it. (2) Why keep TypeScript `strict` mode on — what does it catch, and *when*? (3) What's the difference between how your app runs in development (`dev`) versus production (`start`)? (4) Why split `app.ts` from `server.ts`?

*All boxes ticked and the log written? Then continue. A clean, deliberate skeleton now saves you from a painful re-organise mid-project — and because you've committed to feature/module-based, every chapter from here will line up with your folders instead of fighting them.*

---

Next: the app exists — now stand up the Postgres and Redis it will talk to. → **[Chapter 5 — Your local environment with Docker](05-local-environment-docker.md)**
