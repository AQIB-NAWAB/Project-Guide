# Chapter 19 — Password hashing

The very first row of the threat model you wrote in Chapter 18 was **credential database theft**, and the mindset was **assume breach**: your database *will* leak someday, so the only safe design is one where a stolen copy hands an attacker nothing they can use. Back in Chapter 8 you deliberately left a "password seam" empty in the `accounts` table, promising to fill it the right way once you understood the threat. That day is here — and the right way is emphatically *not* the obvious way.

This is a build chapter, and a careful one, because password storage is the canonical example of something that *looks* fine while being completely broken. You'll add the column, choose a real hashing library, build a tiny hash-and-verify utility, and then *prove* — against your own database — that a stolen row gives an attacker nothing.

## Where we're headed

By the end your `accounts` table has a `password_hash` column (added by a new migration), a vetted hashing library is wired up behind a small utility, and you've verified that the same password hashes to *different* values each time, that verification still works, and that no plaintext password exists anywhere in your database.

## Step 1 — The wrong ladder: plaintext, then encryption, then a fast hash

Walk the tempting options in order — each is better than the last and still wrong, and seeing *why* builds the real solution.

**Plaintext.** Store the password as typed. The moment the database leaks, *every* account is compromised — and because people reuse passwords, so is every other site where they used the same one. Never, under any circumstances.

```
accounts (plaintext — catastrophic)
 email          │ password
 amid@…         │ hunter2          ← one leak and this is everywhere
```

**"We'll encrypt them."** Sounds responsible; it's the wrong tool. Encryption is **reversible** — it exists so you can get the original *back*, which means there's a **key**. And that key almost always lives right next to the database (same server, same config), so the breach that steals your data usually steals the key too. Worse, you *never need to reverse a password* — to check a login you only ever need to verify a guess, not recover the original. Reversibility here is pure downside: all the risk of a recoverable secret, for a capability you don't want.

**Hashing — one-way.** Now the right shape: a **hash** is a one-way fingerprint. You can compute it from a password, but you can't run it backwards to get the password. You store the fingerprint; to check a login you fingerprint the guess and compare. There's no key to steal and nothing to reverse. *But* — not all hashing is equal:

**A fast, general-purpose hash (MD5, SHA-256).** One-way, yes — but built for *speed*, and speed is the enemy here. An attacker holding your leaked hashes can guess **billions of passwords per second** offline on commodity hardware. And without anything extra, the *same* password always produces the *same* hash — so attackers use **rainbow tables** (giant precomputed hash→password lookups), and they can instantly see which of your users share a password. A fast hash alone is barely a speed bump.

Two ideas fix that last rung: **salt**, and **slowness**.

## Step 2 — Salt: defeat precomputation and reuse-spotting

A **salt** is random data, unique per password, mixed in *before* hashing. With it, the same password produces a *different* hash for every user:

```
password "hunter2" hashed WITH a per-user salt
 user A → salt_A + hunter2 → 9f3a…   ┐ same password,
 user B → salt_B + hunter2 → c20b…   ┘ totally different stored hashes
```

This kills two attacks at once. **Rainbow tables become useless** — a precomputed table would have to be rebuilt for every possible salt, which is infeasible. And you **can no longer tell that two users share a password**, because their hashes differ. The salt isn't secret — it's stored right alongside the hash (modern libraries embed it *inside* the hash string) — it just has to be unique and random.

## Step 3 — Slow on purpose: password-specific hashes

Here's the counterintuitive heart of it: a password hash should be **deliberately slow and resource-hungry.** Why would you *want* slow? Because the cost falls almost entirely on the **attacker**, not you. You hash *one* password per login — a few milliseconds, imperceptible. An attacker trying to crack a leaked database must hash *billions* of guesses — and if each hash is made 100,000× slower, their billions-per-second collapses to a crawl. **Slowness is the feature.**

That's what **password-specific algorithms** — **bcrypt**, **scrypt**, **Argon2** — are built for. They're intentionally slow, and the newer ones are **memory-hard** (each hash needs a chunk of memory), which defeats the cheap parallelism of GPUs and custom cracking hardware. Crucially, they expose a tunable **work factor** (cost parameter): you turn the difficulty *up* as computers get faster, so your stored hashes stay expensive to crack for years. (A general hash like SHA-256 has no such dial — it's just fast, forever.)

## Step 4 — Decision: which algorithm?

A legitimate-alternatives choice, so weigh them:

| Algorithm | Notes | Verdict |
|---|---|---|
| **Argon2id** | Won the Password Hashing Competition; memory-hard; current best practice | **Preferred** |
| **bcrypt** | Older, battle-tested, available everywhere, well-understood | **Fine** |
| **scrypt** | Memory-hard, solid | Fine |
| MD5 / SHA-256 / SHA-1 | Fast general hashes — *not* for passwords | **Never** |

**The course requires Argon2id where your stack supports it cleanly, otherwise bcrypt.** Both are correct; pick one and use it everywhere. And the one *absolute* rule, bolded because it has no exceptions: **never implement the hashing yourself — always use a vetted library** (Step 6).

## Step 5 — How verification works without ever reversing

Since you can't un-hash, how does login check a password? You **re-hash the guess and compare**. On login: take the submitted password, hash it with the *stored* salt and parameters, and compare the result to the stored hash. Because modern libraries embed the salt and work factor *inside* the hash string, your utility's surface is just two functions:

```
// the CONTRACT of your password utility — you implement these over the library
hashPassword(plain: string)            → Promise<string>   // returns salt+params+hash, one string
verifyPassword(plain, storedHash)      → Promise<boolean>  // re-hashes the guess and compares
```

One subtlety the library handles for you but you should *know*: the comparison must be **constant-time**, so the server takes the same time whether the first byte was wrong or only the last — otherwise a **timing attack** can leak information about the hash. Never compare hashes with a plain `===`; let the library's `verify` do it.

## Step 6 — Don't roll your own crypto

This deserves its own step because it's the mistake that quietly undoes everything above. Cryptography fails in subtle, invisible ways — a salt that isn't random enough, a comparison that isn't constant-time, a work factor set too low, a homegrown "clever" scheme with a fatal flaw. A vetted library has had these edges found and fixed by experts over years. **Your job is to *use* it correctly, not to invent it.** Start from the library's safe defaults (they choose a sane cost for you), and only tune the work factor later, by measurement. If you ever feel the urge to be clever with hashing, that urge is the bug.

> 💡 **Hint (advanced) — pepper.** For an extra layer, some systems add a **pepper**: a single secret mixed into *every* password, stored *outside* the database (in your Chapter 6 config/env, never in the DB). Then even a full database leak leaves the attacker missing an ingredient that wasn't in the dump. It's optional defense-in-depth, not required here — but it's a clean example of the "assume breach" mindset: keep a secret somewhere the database breach can't reach.

## Step 7 — Do it on your project (hands-on)

Build it and prove it against your real database:

**1. Add the column with a new migration.** Following Chapter 12's golden rule — *never edit an applied migration* — add a **new** migration that adds `password_hash` (text, not null) to `accounts`. (Your dev data will be re-seeded in step 5, so the not-null is fine.) Run `npm run migrate` and confirm the column exists.

**2. Choose and install your hashing library** (Step 4) — Argon2id or bcrypt.

**3. Build the password utility** — a small module exposing `hashPassword` and `verifyPassword` (Step 5's contract) over the library, using its safe defaults. This is the *only* place your app hashes or checks a password.

**4. Prove salt and one-wayness.** Hash a test password **twice** and look at the two results:

```
hashPassword("correct horse battery staple")  → "$argon2id$v=19$...AAA"
hashPassword("correct horse battery staple")  → "$argon2id$v=19$...ZZZ"   # DIFFERENT — the salt at work
```

Two different strings for the *same* password — that's the salt. Now verify:

```
verifyPassword("correct horse battery staple", storedHash)  → true
verifyPassword("wrong password",               storedHash)  → false
```

And look at a stored hash directly: there is no way to read the password out of it. That's the whole point — a leaked row is useless.

**5. Re-seed with hashed passwords.** Update your Chapter 13 seed so every seeded account stores a **hashed** password (run the plaintext dev password through `hashPassword`), not plaintext. Re-run `npm run seed` and inspect `accounts` — you'll see only hashes. Now Chapter 20's login has real accounts to authenticate against, and your database has **zero plaintext passwords**.

> 💡 **Hint — never log the password.** A sneaky leak: logging `req.body` on a register or login request writes the plaintext password straight into your logs. Make sure passwords never reach a log line (you'll formalise logging in Chapter 57). The safest hash in the world doesn't help if the plaintext sits in `stdout`.

> **📖 Mandatory read — before Chapter 20.** Read the **OWASP Password Storage Cheat Sheet** (search that name) and your chosen library's docs on hashing and verifying. *Required: it's the canonical checklist for exactly this chapter, and Chapter 20 wires this utility into real registration and login.*

> **Interesting to read.** Tools like **hashcat** crack billions of fast-hash guesses per second on a single GPU — which is *why* leaked databases of MD5 or unsalted SHA-1 passwords get cracked wholesale within days (LinkedIn 2012, Chapter 18), while a database of properly-salted Argon2 hashes stays expensive to crack for years. Search *"hashcat password cracking benchmarks"* to see the raw speeds that make "slow on purpose" the only sane choice.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 20:

- [ ] `accounts` has a **`password_hash`** column, added via a **new migration** (no applied migration was edited)
- [ ] A **vetted library** (Argon2id or bcrypt) is installed and used through a small `hashPassword` / `verifyPassword` utility — you did **not** implement the algorithm yourself
- [ ] Hashing the **same password twice gives different hashes** (you saw it) — proving per-password salting
- [ ] `verifyPassword` returns **true for the correct password and false for a wrong one** (you tested both)
- [ ] A stored hash **cannot be reversed** to the password; **no plaintext password** exists in the database
- [ ] Your **seed stores hashed passwords**, so the database is clean of plaintext

> **✍️ Log it (mandatory).** In `learning-log/19-password-hashing.md` — **decision** first, then **topics**: **(Decision)** Why store a *hash* rather than the plaintext or an *encrypted* password — what's wrong with encryption specifically here? Which algorithm did you choose and why? **(Topics)** (1) Why is a *slow* hash safer than a *fast* one — who pays the cost? (2) What two attacks does a per-password *salt* defeat? (3) You can't un-hash a password, so how does login verify one — and why does that mean you never needed reversibility? (4) Why is "never roll your own crypto" a hard rule, not a suggestion?

*All boxes ticked and the log written? Then continue. Passwords are now stored so a breach can't expose them — next you put this utility to work in real registration and login, and close the account-enumeration hole while you're there.*

---

Next: with safe password storage in place, build the front door itself — register and log in — without leaking *which* emails exist. → **[Chapter 20 — Register and login](20-register-and-login.md)**
