# Chapter 6 — Configuration & secrets from the environment

Your app boots (Chapter 4) and your services are up (Chapter 5), each reachable at a URL you named — `DATABASE_URL`, `REDIS_URL`. The job now is to get those settings *into* your app the production way. It sounds like plumbing, but configuration done sloppily is where two of the most common production disasters begin: a **secret leaked into Git**, and an app that **boots happily with broken settings** and fails unpredictably hours later. In this chapter you'll walk into both problems on purpose, see why the obvious fixes fall short, and end with one small module that makes both disasters structurally impossible.

We'll go step by step. Don't skip ahead to the final code — the point is to *feel* each problem before you fix it, because that's what makes the final shape obvious instead of arbitrary.

## Where we're headed

By the end you'll have **one typed, validated configuration module** that is the single source of truth for every setting. It reads everything from the **environment** (never hardcoded), **validates it at startup**, and **refuses to boot** with a clear error if anything required is missing or malformed. Secrets will live in a git-ignored `.env`; a committed `.env.example` will document what's needed. Let's build up to that one step at a time.

## Step 1 — Start the way everyone starts (and feel the first problem)

You need a database connection and a secret for signing tokens later. The fastest thing that works is to write the values right where you use them:

```ts
// ❌ the tempting first version — config baked into the code
const db = connect("postgresql://market:localdevpw@localhost:5432/marketplace");
const jwtSecret = "super-secret-signing-key";
```

Run it. It works on your laptop. So what's wrong?

Two things, and both are quiet until they're catastrophic:

- **You just wrote a secret into your source.** The moment you commit, that database password and `jwtSecret` are in Git history *forever* — exactly the leak you've been guarding against since Chapter 4. Deleting the line later doesn't help; history remembers. (We'll see below how fast that gets exploited in the wild.)
- **These values can't change per environment.** Your laptop, a teammate's laptop, staging, and production each need *different* URLs and secrets. A hardcoded string can only be one of them. To deploy, you'd be editing source code — which is precisely how a staging password ends up pointed at the production database.

So hardcoding is out. The values have to come from *outside* the code. That outside place has a standard name: the **environment**.

> The principle here isn't ad-hoc — it's the **Twelve-Factor App** rule: *configuration lives in the environment, never in the code.* The same compiled app runs everywhere; only the environment around it changes.

## Step 2 — Move it to the environment (and feel the second problem)

Node exposes the environment as `process.env`. So the natural next move is to read your values from there, wherever you happen to need them:

```ts
// ❌ better — but scattered, untyped, unvalidated
const port = process.env.PORT;            // the string "3000" — NOT the number 3000
const dbUrl = process.env.DATABASE_URL;   // might be undefined; you find out when it's used
const jwtSecret = process.env.JWT_SECRET; // …and this same pattern, repeated in twenty files
```

No more hardcoded secrets — real progress. But run this in anger and three new problems surface, every one of which bites in *production*, not on your laptop:

- **Everything is a string, or undefined.** `process.env.PORT` is the string `"3000"`, not the number `3000`. `process.env.DATABASE_URL` could be `undefined`. And TypeScript can't catch it — remember Chapter 3, the types are erased at runtime, so TS *believes* these are strings even when they're missing. Unparsed, unchecked values spread bad data through your app.
- **Nothing warns you when a value is missing.** Forget to set `JWT_SECRET` and nothing complains at boot. The app starts *fine* — then crashes deep inside a request, or worse, runs **insecurely**. The failure happens far away from its cause, at the worst possible time.
- **There's no single source of truth.** Twenty scattered `process.env` reads mean twenty places to typo a key, twenty places with no default, and not one file that answers the question "what configuration does this app even need?"

The environment was the right *source*. Reading it ad-hoc, all over the app, is the wrong *method*. Let's fix the method.

## Step 3 — The right approach: one validated module that fails fast

Here's the decision, stated plainly: **exactly one module** reads `process.env`. It validates the whole environment against a schema *at startup*, coerces strings into the right types, and exports a clean, typed `config` object. Everything else in the app imports that object and never touches `process.env` again.

Three properties make this work, and we'll add them one at a time:

1. **Validate** — check every value up front, so a missing or malformed setting is caught immediately.
2. **Fail fast** — if validation fails, the app *refuses to boot*, loudly, pointing at the exact problem.
3. **Type & coerce** — turn `"3000"` into `3000` once, here, so the rest of the app gets real types.

You already have the tool for the validation: **zod**, your validation library from Chapter 3. (If it isn't installed yet, add it now:)

```bash
npm install zod
```

Now write the module. Read it top to bottom — each part maps to one of the three properties above:

```ts
// src/shared/config.ts — the ONLY place in the app that reads process.env
import { z } from 'zod';

// (1) VALIDATE — declare every variable the app needs, and its rules.
const EnvSchema = z.object({
  NODE_ENV:     z.enum(['development', 'test', 'production']).default('development'),
  PORT:         z.coerce.number().int().positive().default(3000),  // (3) COERCE: "3000" → 3000
  DATABASE_URL: z.string().url(),                                  // required; secrets get no default
  REDIS_URL:    z.string().url(),
  JWT_SECRET:   z.string().min(32),                               // required AND long enough to be safe
});

// Validate the WHOLE environment once, here, at startup.
const parsed = EnvSchema.safeParse(process.env);

// (2) FAIL FAST — if anything is missing or malformed, crash NOW, on purpose.
if (!parsed.success) {
  // log the keys and the problem — NEVER the values — then exit
  console.error('✖ Invalid environment configuration', parsed.error.flatten().fieldErrors);
  process.exit(1);
}

// Typed, validated, guaranteed-present. The rest of the app imports THIS.
export const config = parsed.data;
```

That's the whole module. Look at what each rule earns you: `z.coerce.number()` hands you a real number instead of a string; `.url()` rejects a malformed database URL; `.min(32)` rejects a signing secret too short to be safe. And the `safeParse` + `process.exit(1)` is the single most important habit in this chapter — **fail fast**. If the config is wrong, the app doesn't limp along; it stops at the door and tells you exactly what's wrong:

```
$ npm run dev
✖ Invalid environment configuration {
   JWT_SECRET: [ 'String must contain at least 32 character(s)' ],
   DATABASE_URL: [ 'Required' ]
 }
# app exits — fix your .env and try again
```

A noisy crash at boot, in front of *you*, beats a quiet misconfiguration that surfaces as a customer's weird bug three hours later. Every time.

## Step 4 — Use the config everywhere else

With the module in place, the rest of the app gets the easy life. Import `config` and use typed, guaranteed values — no parsing, no `undefined` checks, no `process.env`:

```ts
// ✅ everywhere else in the app
import { config } from './shared/config';

app.listen(config.PORT);                  // a number, guaranteed present
connectToDatabase(config.DATABASE_URL);   // a valid URL, guaranteed present
```

In practice, the two modules that hold live connections to your Docker services read their URLs from `config` — never from `process.env`, never hardcoded:

```ts
// src/shared/db.ts
import { config } from './config';
export const db = createDbClient(config.DATABASE_URL);   // your ORM/client from Chapter 3

// src/shared/redis.ts
import { config } from './config';
export const redis = new Redis(config.REDIS_URL);
```

The rest of the app imports `db` and `redis` from here — one connection each, configured once, validated at the door.

> 💡 **Hint — the one rule that keeps this clean.** Make yourself a rule and never break it: **only `config.ts` may read `process.env`.** If you ever spot `process.env` anywhere else, treat it as a crack in the dam and route it through the config module. (A lint rule can even enforce this for you.)

## Step 5 — Supply the values: `.env` and `.env.example`

The schema *demands* values — so where do they come from in development? From a **`.env`** file at your project root, holding your real local values. It is **git-ignored** (you added it to `.gitignore` in Chapter 4), so it never reaches Git:

```bash
# .env  — your real local values, NEVER committed
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://market:localdevpw@localhost:5432/marketplace
REDIS_URL=redis://localhost:6379
JWT_SECRET=a-long-random-string-of-at-least-32-characters-xxxxxxxx
```

Don't *invent* that `JWT_SECRET` by mashing the keyboard — a signing secret's whole job is to be unguessable, so generate real randomness:

```bash
openssl rand -base64 48
# or, with Node:
node -e "console.log(require('crypto').randomBytes(48).toString('base64'))"
```

But a git-ignored file is invisible to your teammates — clone the repo and there's no `.env`, so the app won't boot. The fix is a second file you *do* commit: **`.env.example`**, the same keys with **placeholder** values and no real secrets. It's the *contract* that tells anyone cloning the repo exactly what to set:

```bash
# .env.example  — committed; keys + placeholders, no real secrets
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://USER:PASSWORD@localhost:5432/DBNAME
REDIS_URL=redis://localhost:6379
JWT_SECRET=set-me-to-a-long-random-string-min-32-chars
```

One more step to make the `.env` actually load: in **development**, its contents need to reach `process.env` before your config module reads them. Either use your framework/runtime's built-in support (e.g. Node's `--env-file=.env` flag in your `dev` script) or a tiny loader library — pick one, wire it into the `dev` script you wrote in Chapter 4.

## Step 6 — How production differs (and what to do if a secret leaks)

You do **not** ship a `.env` file to production. Instead the platform injects the real values as actual environment variables, and your *same* config module validates them identically — dev, staging, and production all run the one schema; only the values differ. That's the payoff of centralising: there is exactly one place that defines what valid configuration means, and it guards every environment the same way.

And if a secret ever does slip into Git despite all this? Deleting the line is **not** enough — it's in the history, and history is forever. You must **rotate** it: change the secret at its source (issue a new database password, a new signing key) so the leaked value becomes worthless. Treat a leaked secret as already-compromised, every time.

> **📖 Mandatory read — before Chapter 7.** Read the **Twelve-Factor App** chapter on **Config** (search: *"twelve factor app config"*), and skim your validation library's docs on parsing/coercing environment variables. *Required: the rest of the course assumes config flows through one validated module — understand the principle, not just the snippet.*

> **Interesting to read.** Committed credentials get found *fast*. GitHub runs **secret scanning** that detects them automatically and can notify the provider to auto-revoke them, often within minutes of a push. Search *"GitHub secret scanning"*. The fact that this service has to exist at industrial scale tells you how routine — and how exploited — the mistake is.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 7:

- [ ] A single **`config` module** reads `process.env`, validates it with a schema, and exports a typed config object
- [ ] The app **refuses to boot** with a clear, specific error when a required variable is missing or malformed (test it: blank out `JWT_SECRET` and confirm the crash)
- [ ] Types are coerced (e.g. `PORT` is a **number**, not a string) and constraints enforced (e.g. `JWT_SECRET` length)
- [ ] **Nothing outside `config.ts` reads `process.env`** directly — the app uses `config.*`
- [ ] `DATABASE_URL` and `REDIS_URL` flow through config, and the app connects to the Docker services using them
- [ ] `.env` is git-ignored and loaded in development; **`.env.example`** is committed with keys and placeholders (no real secrets)
- [ ] No secret is in the source or in Git history

> **✍️ Log it (mandatory).** In `learning-log/06-config-and-secrets.md`: (1) Why does configuration belong in the environment rather than in code — give the two failures it prevents. (2) Why validate config and **fail fast at startup** instead of reading `process.env` where you need it? Describe the bad outcome fail-fast avoids. (3) What is `.env.example` *for*, and why is it safe to commit when `.env` is not?

*All boxes ticked and the log written? Then continue. With config validated at the door, every chapter after this can trust its settings instead of guarding against them.*

---

Next: the app is configured and connected — give it its first real endpoint, the one your deploy platform will watch. → **[Chapter 7 — The health check endpoint](07-health-check.md)**
