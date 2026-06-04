# Chapter 6 — Configuration & secrets from the environment

Your app boots (Chapter 4) and your services are up (Chapter 5), each reachable at a URL you named — `DATABASE_URL`, `REDIS_URL`. Now you wire those into the app *the production way*. This sounds like plumbing, but configuration done sloppily is where two of the most common production disasters start: a **secret leaked into Git**, and an app that **boots happily with broken settings** and then fails unpredictably at 3 a.m. We'll make both structurally impossible.

## The point of this chapter

One **typed, validated configuration module** that is the single source of truth for every setting. It reads everything from the **environment** (never hardcoded), **validates it at startup**, and **refuses to boot** with a clear error if anything required is missing or malformed. Secrets live in a git-ignored `.env`; a committed `.env.example` documents what's needed.

## Decision 1 — Where configuration lives: in the code, or in the environment?

The tempting approach is to put settings right where they're used:

```ts
// ❌ the tempting version — config baked into the code
const db = connect("postgresql://market:localdevpw@localhost:5432/marketplace");
const jwtSecret = "super-secret-signing-key";
```

It works on your laptop. Here's why it disappoints, structurally:

- **It leaks secrets.** That `jwtSecret` and database password are now in your source, and the moment you commit, they're in Git history forever — exactly the leak you've been guarding against since Chapter 4. (More on how fast that gets exploited below.)
- **It can't change per environment.** Your laptop, a teammate's laptop, staging, and production all need *different* database URLs and secrets. Hardcoded values can't be all of those at once — you'd be editing source to deploy, which is how wrong values reach production.

The professional rule, drawn straight from the **Twelve-Factor App** methodology, is: **configuration lives in the environment, never in the code.** The same compiled app runs everywhere; only the environment around it changes. Your local values come from a `.env` file; production injects real values as actual environment variables.

## Decision 2 — Reading `process.env` is not enough

So config comes from `process.env`. The tempting next step is to read it directly, wherever you need it:

```ts
// ❌ scattered, untyped, unvalidated
const port = process.env.PORT;            // a string | undefined — NOT a number
const dbUrl = process.env.DATABASE_URL;   // could be undefined; you find out when it's used
// …repeated in twenty files, each a place a typo or a missing var hides until runtime
```

Three quiet problems, every one of which bites in production:

- **Everything is a string — or undefined.** `process.env.PORT` is the string `"3000"`, not the number `3000`; `process.env.DATABASE_URL` might be `undefined`. TypeScript can't save you here (remember Chapter 3 — types are erased), so unparsed, unchecked env values spread bad data through your app.
- **No validation, no early warning.** Forget to set `JWT_SECRET` and nothing complains at boot — your app starts *fine* and then either crashes deep inside a request or, worse, runs **insecurely**. The failure happens far from the cause.
- **No single source of truth.** Twenty scattered `process.env` reads mean twenty places to typo a key, twenty places with no default, and no one file that answers "what config does this app need?"

## The right approach — one validated config module that fails fast

Centralise it. Have **exactly one module** read `process.env`, validate the whole thing against a schema at startup, coerce types, and export a clean, typed config object. Everything else in the app imports *that* and never touches `process.env` again. With zod (your validation library from Chapter 3), the shape is:

```ts
// src/shared/config.ts — the ONLY place that reads process.env
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV:     z.enum(['development', 'test', 'production']).default('development'),
  PORT:         z.coerce.number().int().positive().default(3000),  // string "3000" → number 3000
  DATABASE_URL: z.string().url(),                                  // required; secrets get no default
  REDIS_URL:    z.string().url(),
  JWT_SECRET:   z.string().min(32),                               // required AND long enough to be safe
});

// Validate the WHOLE environment once, at startup. If it's wrong, crash HERE — on purpose.
const parsed = EnvSchema.safeParse(process.env);
if (!parsed.success) {
  // log which keys are missing/invalid (the keys and the problem — never the values), then exit
  console.error('✖ Invalid environment configuration', parsed.error.flatten().fieldErrors);
  process.exit(1);
}

export const config = parsed.data;   // typed, validated — the rest of the app imports THIS
```

Read what this buys you. **`z.coerce.number()`** turns the string into a real number. **`.url()` and `.min(32)`** reject a malformed database URL or a too-short signing secret. And the `safeParse` + `process.exit(1)` is the most important habit in the whole chapter: **fail fast.** If the config is wrong, the app *refuses to start*, loudly, pointing at the exact problem:

```
$ npm run dev
✖ Invalid environment configuration {
   JWT_SECRET: [ 'String must contain at least 32 character(s)' ],
   DATABASE_URL: [ 'Required' ]
 }
# app exits — fix your .env and try again
```

A noisy crash at boot, in front of *you*, is infinitely better than a quiet misconfiguration that surfaces as a weird bug for a customer later. Everywhere else, config is now typed and guaranteed:

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

The rest of the app imports `db` and `redis` from here — one connection, configured once, validated at the door.

> 💡 **Hint — the rule that keeps this clean.** Make one rule for yourself and never break it: **only `config.ts` may read `process.env`.** If you ever find `process.env` anywhere else, that's a leak in the dam — route it through the config module. (A lint rule can even enforce this for you.)

## The two files: `.env` and `.env.example`

Your **`.env`** holds your real local values and is **git-ignored** (you added it to `.gitignore` in Chapter 4):

```bash
# .env  — your real local values, NEVER committed
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://market:localdevpw@localhost:5432/marketplace
REDIS_URL=redis://localhost:6379
JWT_SECRET=a-long-random-string-of-at-least-32-characters-xxxxxxxx
```

Don't *invent* that `JWT_SECRET` by mashing the keyboard — a signing secret's entire job is to be unguessable, so generate real randomness:

```bash
openssl rand -base64 48
# or, with Node:
node -e "console.log(require('crypto').randomBytes(48).toString('base64'))"
```

But a git-ignored file is invisible to teammates — so you also commit **`.env.example`**: the same keys with **placeholder** values and no real secrets. It's the *contract* that tells anyone cloning the repo exactly what to set:

```bash
# .env.example  — committed; keys + placeholders, no real secrets
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://USER:PASSWORD@localhost:5432/DBNAME
REDIS_URL=redis://localhost:6379
JWT_SECRET=set-me-to-a-long-random-string-min-32-chars
```

In **development**, load `.env` into `process.env` at startup (via your framework's support or a small loader). In **production**, you do *not* ship a `.env` file — the platform injects the real values as actual environment variables, and your same config module validates them identically. (If a secret ever does slip into Git, deleting it isn't enough — it's in the history, so you must **rotate** it: change the secret at its source.)

> **📖 Mandatory read — before Chapter 7.** Read the **Twelve-Factor App** chapter on **Config** (search: *"twelve factor app config"*), and skim your validation library's docs on parsing/coercing environment variables. *Required: the rest of the course assumes config flows through one validated module; understand the principle, not just the snippet.*

> **Interesting to read.** Committed credentials get found *fast* — GitHub runs **secret scanning** that detects them and can notify providers to auto-revoke them, often within minutes of a push. Search *"GitHub secret scanning"*. The fact that this service has to exist at industrial scale tells you how routine — and how exploited — the mistake is.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 7:

- [ ] A single **`config` module** reads `process.env`, validates it with a schema, and exports a typed config object
- [ ] The app **refuses to boot** with a clear, specific error when a required variable is missing or malformed (test it: blank out `JWT_SECRET` and confirm the crash)
- [ ] Types are coerced (e.g. `PORT` is a **number**, not a string) and constraints enforced (e.g. `JWT_SECRET` length)
- [ ] **Nothing outside `config.ts` reads `process.env`** directly — the app uses `config.*`
- [ ] `DATABASE_URL` and `REDIS_URL` flow through config, and the app connects to the Docker services using them
- [ ] `.env` is git-ignored; **`.env.example`** is committed with keys and placeholders (no real secrets)
- [ ] No secret is in the source or in Git history

> **✍️ Log it (mandatory).** In `learning-log/06-config-and-secrets.md`: (1) Why does configuration belong in the environment rather than in code — give the two failures it prevents. (2) Why validate config and **fail fast at startup** instead of reading `process.env` where you need it? Describe the bad outcome fail-fast avoids. (3) What is `.env.example` *for*, and why is it safe to commit when `.env` is not?

*All boxes ticked and the log written? Then continue. With config validated at the door, every chapter after this can trust its settings instead of guarding against them.*

---

Next: the app is configured and connected — give it its first real endpoint, the one your deploy platform will watch. → **[Chapter 7 — The health check endpoint](07-health-check.md)**
