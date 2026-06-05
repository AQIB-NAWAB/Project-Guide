# Chapter 20 — Register and login

You can store a password so a breach can't expose it (Chapter 19). Now you build the two doors that password is *for*: **registration** (becoming an account) and **login** (proving you are one). These are small endpoints with a sharp edge — login is one of the most attacked surfaces in any app, and the naive version leaks information that helps attackers before they've even guessed a password. So you'll build them carefully, and close the **account-enumeration** hole from your threat model while you're at it.

This is a build chapter that puts the Chapter 19 utility to work, runs through your validation (Chapter 16) and error contract (Chapter 17), and creates the identity + profile rows you modelled back in Chapter 8.

## Where we're headed

By the end you'll have `POST /auth/register` (validates input, rejects a duplicate email with `409`, hashes the password, creates the account and its role profile) and `POST /auth/login` (verifies the password and tells you the credentials are good) — with **uniform responses** that never reveal which emails exist. Logging in won't yet *keep* you logged in; that cliffhanger is Chapter 21.

## Step 1 — Register: turn input into an account

Registration takes an email, a password, and a role, and creates an account. Lean on what you've already built:

- **Validate the body** (Chapter 16) with a schema: a well-formed email, a password meeting your policy (Step 4), a role that's one of your allowed values. Reject malformed input with the consistent `400`.
- **Check the email isn't taken.** Your `accounts.email` is `UNIQUE` (Chapter 11), so the database *will* reject a duplicate — but you want a clean **`409 Conflict`** through your error contract (Chapter 17), not an ugly constraint crash. Check first, and throw `ConflictError` if it exists.
- **Hash the password** with your Chapter 19 `hashPassword` utility — never store what was typed.
- **Create the account *and* its role profile.** Recall Chapter 8: identity lives in `accounts`, role data in `vendor_profiles` / `customer_profiles`. Registering a vendor creates both rows. Because they must both succeed or neither should, this is a natural place for a transaction — you'll formalise transactions in Chapter 47; for now, just know these two writes belong together.
- **Respond `201 Created`** — and return the new account *without* the password hash. Never send the hash back.

```
POST /auth/register
{ "email": "maker@example.com", "password": "...", "role": "vendor" }
  → 201  { "id": "uuid", "email": "maker@example.com", "role": "vendor" }   // no hash, ever
  → 409  { "error": { "type": "conflict", "message": "Email already registered" } }
  → 400  { "error": { "type": "validation_error", "fields": { ... } } }
```

## Step 2 — Login: verify without ever revealing the password

Login takes an email and password and checks them. The flow is: find the account by email, then use your Chapter 19 `verifyPassword(submitted, account.password_hash)`. If it returns true, the credentials are good. You **never** decrypt or compare plaintext — you re-hash the guess and let the library compare (Chapter 19, Step 5).

But *how you respond when it fails* is where security lives — which is the whole next step.

## Step 3 — Close the enumeration hole (the security beat)

Here's the naive login, and the leak hiding in it. It returns different errors for different failures:

```
// ❌ leaks which emails exist
account not found        → 404 "No account with that email"
wrong password           → 401 "Incorrect password"
```

That's **account enumeration** (threat model, Chapter 18). An attacker submits a list of emails and reads the *difference* in your responses — "no account" vs "incorrect password" tells them exactly which emails are real. Now they have a confirmed list of accounts to target with password guessing, phishing, or credential-stuffing.

The fix is **uniform responses**: the same status, the same message, for *both* "no such email" and "wrong password."

```
// ✅ reveals nothing
no such email     → 401 "Invalid email or password"
wrong password    → 401 "Invalid email or password"
```

There's a subtler tell, too: **timing.** If "email not found" returns instantly but "wrong password" takes time (because it ran the slow hash), an attacker can *time* your responses to learn which emails exist. The defence: when the email isn't found, still run a `verifyPassword` against a **dummy hash** so the failure path takes about the same time as a real one. Same answer, same shape, same timing — the attacker learns nothing.

> Note the tension with *registration*: it inherently reveals a taken email via the `409`. That's a known, accepted trade-off (a signup form has to tell you the email's in use), and it's partly why robust systems lean on email verification (Chapter 23) so an unverified "registration" can't confirm much. Login and password-reset, which attackers probe far more, are where uniform responses matter most.

## Step 4 — A sane password policy

Don't impose the old "must have an uppercase, a number, and a symbol" rules — modern guidance (NIST) is clear that **length beats complexity**: a long passphrase is stronger and more memorable than `P@ss1!`. So require a reasonable **minimum length** (e.g. 8–12+), allow (and don't truncate) long passwords and spaces, and — optionally, as a nice touch — reject passwords known to be in breach corpuses (the *Have I Been Pwned* range API lets you check without sending the full password). Keep it user-friendly; the slow hash (Chapter 19) is doing the heavy lifting, not arbitrary character rules.

## Step 5 — Do it on your project (hands-on)

Build both endpoints and attack them:

**1. `POST /auth/register`** — validate, check-then-`409` on duplicate email, hash with your utility, create the account + role profile, return `201` without the hash.

**2. `POST /auth/login`** — validate, look up by email, `verifyPassword`, and return success on a match. (What success *returns* gets real tokens in Chapter 21; for now a `200` with the user is fine.)

**3. Make responses uniform** — same `401` and message for unknown-email and wrong-password, and run a dummy `verifyPassword` on the unknown-email path to level the timing.

**4. Test it like an attacker:**

```bash
# register
curl -i -X POST localhost:3000/auth/register -d '{"email":"m@x.com","password":"a-long-passphrase","role":"vendor"}' -H 'content-type: application/json'   # → 201
curl -i -X POST localhost:3000/auth/register -d '{"email":"m@x.com","password":"a-long-passphrase","role":"vendor"}' -H 'content-type: application/json'   # → 409 (duplicate)

# login — note both failures look IDENTICAL
curl -i -X POST localhost:3000/auth/login -d '{"email":"m@x.com","password":"a-long-passphrase"}' -H 'content-type: application/json'   # → 200
curl -i -X POST localhost:3000/auth/login -d '{"email":"m@x.com","password":"WRONG"}'            -H 'content-type: application/json'   # → 401 "Invalid email or password"
curl -i -X POST localhost:3000/auth/login -d '{"email":"nobody@x.com","password":"whatever"}'    -H 'content-type: application/json'   # → 401 "Invalid email or password"  (same!)
```

The last two responses being indistinguishable — in shape *and* roughly in timing — is the whole point. Your seeded accounts (now with hashed passwords from Chapter 19) are real accounts you can log into.

> 💡 **Hint — the response is a place secrets leak.** Return the account *without* `password_hash`, and make sure nothing about the failure path differs between "unknown email" and "wrong password" — not the status, not the message, not the timing, not even a stray header. Enumeration lives in the differences.

> **📖 Mandatory read — before Chapter 21.** Read the **OWASP Authentication Cheat Sheet** sections on **login** and **account enumeration / generic error messages** (search those terms). *Required: it's the canonical list of exactly the traps this chapter avoids.*

> **Interesting to read.** **Credential stuffing** — replaying username/password pairs leaked from *other* sites' breaches — is one of the most common attacks on login endpoints, and account enumeration is its reconnaissance step: confirm which emails exist here, then stuff. Search *"credential stuffing attack"* to see why denying attackers a confirmed account list is a real, not theoretical, defence.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 21:

- [ ] `POST /auth/register` validates input, **hashes** the password (no plaintext stored), and creates the **account + role profile**, returning `201` **without** the hash
- [ ] A duplicate email returns a clean **`409`** through your error contract, not a constraint crash
- [ ] `POST /auth/login` verifies the password with your Chapter 19 utility (never compares plaintext)
- [ ] **Unknown email and wrong password return the identical `401`** — same shape, same message
- [ ] The unknown-email path runs a **dummy verify** so its timing matches the real path
- [ ] A sensible **length-based** password policy is enforced; long passphrases are allowed

> **✍️ Log it (mandatory).** In `learning-log/20-register-and-login.md` — **decision** first, then **topics**: **(Decision)** Why return an *identical* error for "unknown email" and "wrong password" — what does an attacker gain if they differ? Why check for a duplicate email yourself and throw `409` instead of letting the database constraint fire? **(Topics)** (1) What is account enumeration, and how do uniform responses *and* uniform timing each defend against it? (2) Why is the password hashed at registration and never returned in any response? (3) Why does length beat character-complexity in a password policy? (4) Why do creating the `accounts` row and the role-profile row belong together as one unit?

*All boxes ticked and the log written? Then continue. A user can register and prove who they are — but the moment the response is sent, your server forgets them. Next you give it a memory.*

---

Next: login works, but the next request is anonymous all over again — HTTP forgot you. Time to make the server remember who's calling. → **[Chapter 21 — Access tokens](21-access-tokens.md)**
