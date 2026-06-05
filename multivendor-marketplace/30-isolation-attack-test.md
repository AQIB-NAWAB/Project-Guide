# Chapter 30 — The isolation attack test

You've proven isolation by hand with curl (Chapters 28–29) — vendor A can't read or touch vendor B's data. But a manual check passes *once*, on the day you run it. Isolation is a security property that must hold *forever*, across every future change: the day someone refactors a query and drops a scope, or adds a new endpoint and forgets the ownership check, your marketplace springs a leak — and nobody notices until a vendor sees another's orders. A manual check can't catch that. **A test can.**

This chapter writes an automated **attack test**: code that *plays the attacker*, attempts the cross-tenant access you just defended against, and asserts it's blocked — so the day someone reintroduces an IDOR hole, the build goes red instead of shipping the breach. It's also your first test in this course, so you'll set up the testing foundation you'll build on for the rest of the project.

## Where we're headed

By the end you have a test runner wired up against a test database, an integration test that creates two vendors and asserts vendor A is blocked from reading/editing/deleting vendor B's data (and *can* act on their own), and you've watched the test catch a deliberately reintroduced bug.

## Step 1 — Why a test, not another manual check

The difference is *when* the check runs. Manual verification happens when you remember to do it; a test happens on **every commit**, automatically, forever. For a security property that's the entire point: you're not testing that isolation works *today* — you proved that by hand. You're building a **regression guard** that re-proves it on every future change, catching the well-meaning refactor that quietly removes a scope.

This is the **negative test** mindset, and it's distinct from ordinary testing. Most tests assert the good thing *happens*; a security test asserts the bad thing is *prevented* — that the attack returns `404`, that the cross-tenant edit affects zero rows. You are encoding "this must never be allowed" as something a machine checks.

## Step 2 — The testing foundation (set up once)

You need three things, and they'll serve the whole rest of the course:

- **A test runner** — your stack's standard (e.g. the one your framework recommends). Add a `test` script (alongside `migrate`, `seed` from earlier chapters).
- **A test database** — *separate* from your dev database, so tests can create and wipe data freely without touching your work. Point it at a different `DATABASE_URL` (your Chapter 6 config makes this a one-line switch via the environment), and run your migrations against it so it has the real schema.
- **A clean-state strategy** — each test (or test run) starts from a known state: migrate the test DB, and reset relevant tables between tests (a transaction rolled back per test, or a truncate-and-seed). Deterministic tests need a deterministic starting point.

```
// package.json
"scripts": { "test": "...run the test runner against the TEST database..." }
```

This is **integration testing**: the test drives your real API (real routes, real middleware, real database) end to end — because isolation is a property of the *whole* request path (auth → scope → query), not any single function, so you must test it through the front door.

## Step 3 — The attack scenario

The test embodies the exact attack from Chapters 28–29. In code:

1. **Arrange two tenants.** Create vendor A and vendor B (register or seed), each with a store and at least one product. Log both in to get their access tokens.
2. **Attack as A against B's objects.** Using A's token, attempt every cross-tenant operation on B's product id:
   - `GET /products/<B's id>` (read) → expect **`404`**
   - `PATCH /products/<B's id>` (edit) → expect **`404`**
   - `DELETE /products/<B's id>` (delete) → expect **`404`**
   - `GET /vendor/products` as A → assert **B's products are absent**
3. **Confirm the control case.** As A, act on A's *own* product → expect success (`200`). This proves the test isn't passing simply because *everything* is blocked.

Each assertion is a line in the sand: cross-tenant access must be `404`, own-tenant access must work.

## Step 4 — Do it on your project (hands-on)

**1. Set up the runner and test DB** (Step 2): a `test` script, a separate `DATABASE_URL` for tests, migrations applied to it, and a reset-between-tests strategy.

**2. Write the isolation test** following Step 3 — two vendors, A attacks B's product across read/edit/delete (assert `404`), A's own list excludes B's products, A's own edits succeed.

**3. Run it green:**

```bash
npm test    # → the isolation test passes: cross-tenant blocked, own-tenant allowed
```

**4. Prove the test actually guards (the crucial step).** Temporarily **break** isolation — remove the `AND store_id = ...` scope from your product edit query (reintroducing the IDOR bug) — and run the test again:

```bash
npm test    # → the isolation test FAILS: A could edit B's product. The guard caught it.
```

Watch it go red, then **restore the scope** and confirm it's green again. A test you've never seen fail is a test you don't trust; you just confirmed this one genuinely detects a real regression.

**5. Wire it to run automatically.** Make `npm test` part of your routine (and your future CI, Chapter 60) so this attack runs on every change forever.

> 💡 **Hint — test the *negative* explicitly, and keep the control.** It's tempting to only assert "A can edit A's product." But the security value is entirely in the assertions that A *cannot* touch B's. Always include both: the negative cases (the attack is blocked) prove isolation; the positive control (own access works) proves the negatives aren't passing by accident because the whole endpoint is broken.

> **📖 Mandatory read — before Chapter 31.** Read your test runner's **getting-started** guide and a short piece on **integration vs unit testing** (search *"integration testing API"*). *Required: this test harness is the foundation you'll reuse for the rest of the course's features.*

> **Interesting to read.** Security regression tests are why mature teams can refactor fearlessly — the test suite re-runs the old attacks on every commit, so a reintroduced IDOR or auth bypass fails the build instead of reaching production. Search *"security regression testing"* to see "encode the attack as a test" treated as standard practice.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 31:

- [ ] A **test runner** is set up with a `test` script and a **separate test database** running your migrations
- [ ] An **integration test** creates two vendors and asserts A is blocked (`404`) from **reading, editing, and deleting** B's product, and that A's list **excludes** B's products
- [ ] The test includes a **positive control**: A *can* act on their own product
- [ ] You **broke isolation on purpose and watched the test fail**, then restored it and watched it pass
- [ ] The test runs with `npm test` and is ready to run on every change

> **✍️ Log it (mandatory).** In `learning-log/30-isolation-attack-test.md` — **decision** first, then **topics**: **(Decision)** Why encode isolation as an automated test instead of relying on the manual checks from Chapters 28–29? Why use an *integration* test through the API rather than a unit test of one function? **(Topics)** (1) What is a "negative test," and why is it the heart of a security test? (2) Why does the test need a positive control case? (3) Why did deliberately breaking the scope and watching the test fail matter? (4) Why must the test database be separate from your dev database?

*All boxes ticked and the log written? **That completes multi-tenant isolation** — your marketplace is now genuinely sealed per vendor, and a test keeps it that way. Next you build the experience that sits on top of this secure foundation: the vendor's own world.*

---

Next: the data is secure and isolated — now build the vendor's actual journey, starting with opening their store. → **[Chapter 31 — Vendor onboarding](31-vendor-onboarding.md)**
