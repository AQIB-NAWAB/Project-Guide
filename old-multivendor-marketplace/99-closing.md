# Closing — Ship, document, prove it

Your marketplace is live (Chapter 60). Take a moment to see how far you've come: you started with an empty repository and a question about which database to use, and you finished with a deployed, secure, multi-tenant marketplace that real makers could sell on — built one careful decision at a time, each understood rather than copied. That journey *is* the achievement. This closing chapter turns it into something that opens doors: documented, defensible, and ready to prove.

A deployed app you can't *explain* is half the value. The point of this course was never just to produce a marketplace — it was to make you someone who can build one and **account for every choice in it**. So the final work is to package the project and prepare to defend it.

## 1. Document it

A project nobody can understand is a project nobody will trust. Produce the documentation that lets a stranger — a reviewer, an interviewer, your future self — grasp what you built and why:

- **An architecture diagram** — the API, the worker, Postgres, Redis, object storage, and how a request and a job flow through them. (You've built the mental model across the course; draw it.)
- **The ERD** — your data model (you sketched it in the data module; finalize it).
- **Key user flows** — register → verify → onboard → list a product; browse → cart → checkout → confirmation; the per-vendor split and payout.
- **The decision records you already wrote** — `SCHEMA-DECISIONS.md`, `THREAT-MODEL.md`, `AUTHORIZATION.md`, `ISOLATION.md`, `IMAGES.md`, `ASYNC.md`. These aren't busywork; together they're a *design dossier* most portfolios completely lack. Link them from your README.
- **Challenges and how you solved them** — the N+1 you measured and killed, the IDOR you closed and tested, the checkout you made atomic and idempotent. The *problems* are the interesting part.
- **A short demo video** — two minutes walking through the live app. Reviewers watch before they read.

Your **`learning-log/`** folder — the answers you wrote after every chapter — is the raw material for all of this, and the single best evidence that you understood what you built rather than copied it.

## 2. Write the case study

Turn the project into a portfolio piece — a blog post, a LinkedIn write-up, a README that reads like a story. Not "I built a marketplace with Node and Postgres," but the *engineering*: "How I kept one vendor from ever seeing another's data — and wrote a test that proves it." "Why every order in my marketplace remembers the price it was sold at." "How I moved slow work off the request path with a job queue." Lead with a problem and a decision; that's what makes other engineers stop scrolling. This is build-in-public, and it compounds — the write-up often opens more doors than the code.

## 3. Raise the bar

Pick **one** feature or improvement that stretches you — and that **respects the core invariants** you built. The rule: it must not break isolation, snapshots, atomicity, or the security model. Some options, each a real extension:

- **Multiple stores per vendor** — relax the one-store rule (Chapter 31), and watch how cleanly it goes *because* you modelled the store as its own table (Chapter 8).
- **Product reviews and ratings** — a new entity with its own ownership and moderation rules.
- **Postgres Row-Level Security** — add the database-enforced isolation backstop you noted in Chapter 27.
- **Real payments** — integrate a payment provider at the checkout transaction (Chapter 47), idempotency keys and all (Chapter 48) — the natural next layer on the payout reporting you built.
- **A dedicated search engine**, multi-currency, an admin panel, inventory/stock tracking, order status workflows with vendor notifications (more async jobs).

Choose one, build it on top of the foundation, and notice how the *quality* of the foundation makes the extension easy or hard — that's the clearest proof that the early decisions mattered.

## 4. The evaluation

Here's how your understanding gets tested: **you'll walk through how the marketplace works, and what would break it.** Be ready to open any part of the system and explain — in your own words, not the guide's — *why* it's built the way it is, and what the alternative would have cost:

- Why is a password hashed and not encrypted? What does the salt do?
- Trace a multi-vendor checkout. Where exactly would it corrupt without a transaction? Without idempotency?
- A vendor changes a product's price. Which past orders change, and why?
- How does one vendor's request get blocked from another's data — at how many layers?
- Why did you choose a job queue over event-driven architecture? Cursor over offset pagination? UUIDs over sequential ids?

If you can answer those cold — and after writing a learning log for all sixty chapters, you can — you didn't just finish a tutorial. You escaped tutorial hell. You can build a real system *and explain every decision in it*, which is exactly what separates someone who follows tutorials from someone who ships software.

## The whole climb, in one view

Look back at the ladder you climbed:

- **Week 1 — Foundations:** config, Docker, a self-defending data model (migrations, constraints, snapshots), validated input, an API that speaks HTTP correctly.
- **Week 2 — Identity & isolation:** hashing, tokens with refresh and revocation, email verification, role-based access, and multi-tenant isolation *proven by a test*.
- **Week 3 — Performance & commerce:** a fast, searchable, cached catalogue (N+1, pagination, indexing), images in object storage, and an atomic, idempotent multi-vendor checkout with honest snapshotted prices.
- **Week 4 — Async, resilience & ship:** a job queue with retries and idempotency, scheduled batch payouts, rate limiting, structured logging, graceful shutdown, hardening, and a live HTTPS deploy.

Every rung was small. Every rung earned the next. And you understood each one before you built it — which was the entire point.

---

**You're done.** You built a production marketplace and you can defend every line of reasoning in it. Now go write the case study, ship the bar-raiser, and tell people what you made — you've earned it.

*Back to the start: [Chapter 1 — Introduction](01-introduction.md)*
