# Chapter 23 — Email verification

Registration and login are solid (Chapters 20–22), but there's a quiet assumption underneath them: that the email on each account actually belongs to the person who registered. Right now it doesn't have to. Anyone can register `someone-elses@email.com`, and worse for a marketplace, a spammer can open a *vendor store* under an address they don't own — fake shops, no accountability, your domain's reputation damaged when their "order confirmations" bounce around. That's the **fake/unverified accounts** row on your threat model (Chapter 18).

The fix is **email verification**: prove the registrant controls the address before the account is fully trusted. It's a small flow with a familiar shape — a single-use, expiring, *hashed* token (Chapter 19's lessons return yet again) — plus a clean decision about what "unverified" is and isn't allowed to do.

## Where we're headed

By the end, registering creates an **unverified** account and issues a one-time verification token; a `POST /auth/verify` endpoint confirms it; certain protected actions (like opening a store or publishing a product) **require a verified account**; and you can resend the token. Actually *sending* the email is stubbed for now — real delivery, moved onto a queue, is Chapter 51.

## Step 1 — Don't fully activate on registration

The principle: registration creates an account, but in an **unverified** state, and verification is a separate step the user completes by proving they received something at that address. So you add a flag to identity — `email_verified` on `accounts` — defaulting to `false`, and you flip it true only when they confirm.

Add it with a new migration (Chapter 12):

```
accounts  (add via new migration)
 email_verified  (boolean, not null, default false)
```

A newly registered user can log in (they have valid credentials) but is *not yet trusted* for sensitive actions until that flag is true.

## Step 2 — The verification token: single-use, expiring, hashed

To prove control of the email, you send a secret to that address and ask the user to return it. That secret is a **verification token** — and it's the password/refresh-token pattern a third time:

- **Random and high-entropy** — unguessable, so an attacker can't just submit tokens until one works.
- **Expiring** — valid for, say, 24 hours, then dead. A leaked or forgotten token doesn't linger forever.
- **Single-use** — consumed the moment it verifies, so it can't be replayed.
- **Stored hashed** — like refresh tokens (Chapter 22), you keep the *hash*, so a database leak doesn't hand over working verification tokens.

Store it much like refresh tokens — a small table (or columns) holding `account_id`, the token **hash**, and `expires_at`. On registration you generate the raw token, store its hash, and put the raw token in a verification link (`/verify?token=...`) sent to the email.

## Step 3 — Sending the email (stubbed today, queued later)

You need to deliver that link to the user's inbox — but real email delivery (an SMTP provider, deliverability, retries) is a Week-4 concern, and sending it *synchronously inside the registration request* is exactly the kind of slow external call you'll learn to move onto a background queue in Chapter 51. So for now, **stub the delivery**: in development, simply **log the verification link** to your console (or write it to a dev mailbox tool). The flow is complete and testable; only the transport is a placeholder.

> This is an honest seam, not a shortcut: the verification *logic* is real and done; Chapter 51 swaps the stub for a real provider and moves the send to a worker so a slow email never delays a user's signup. Note it as a `// TODO: send via queue (Ch51)`.

## Step 4 — Gate the actions that need a real email

Verification only matters if *something* depends on it. Decide which actions require a verified account — for this marketplace, the sensible line is: **anything that creates public, money- or trust-bearing presence** requires verification. Opening a store and publishing products should require it; merely browsing as a logged-in customer need not.

So a verified-only action checks the flag and, if it's false, returns **`403 Forbidden`** through your error contract (Chapter 17) with a clear "verify your email first" message. (This is your first taste of authorization-by-account-state — a natural lead-in to the authorization module that starts in Chapter 25.)

## Step 5 — Do it on your project (hands-on)

**1. Migrate `email_verified`** onto `accounts` (default false) and add your verification-token store. `npm run migrate`.

**2. Issue a token on registration.** Update `POST /auth/register` (Chapter 20) to generate a verification token, store its **hash** with a 24-hour expiry, and **log the verification link** in dev. The account is created `email_verified = false`.

**3. Build `POST /auth/verify`.** It takes the raw token, hashes it, finds the matching unexpired record, sets `email_verified = true`, and **consumes** the token (delete it / mark used). A bad or expired token returns a clean `400`/`410`.

**4. Add a resend endpoint** — `POST /auth/resend-verification` — that issues a fresh token (and invalidates the old) for a logged-in unverified user.

**5. Gate a real action.** Make "publish a product" (or open a store) require `email_verified`. Return `403` if not.

**6. Walk the whole flow:**

```bash
# register → account created unverified; grab the logged verification link
curl -i -X POST localhost:3000/auth/register -d '{"email":"v@x.com","password":"a-long-passphrase","role":"vendor"}' -H 'content-type: application/json'   # → 201, link logged to console

# try the gated action while unverified → blocked
curl -i -X POST localhost:3000/products -H '...auth...' -d '{...publish...}'    # → 403 "verify your email first"

# verify with the token from the logged link
curl -i -X POST localhost:3000/auth/verify -d '{"token":"<from the log>"}' -H 'content-type: application/json'   # → 200

# now the gated action works
curl -i -X POST localhost:3000/products -H '...auth...' -d '{...publish...}'    # → 201
```

Unverified → blocked → verify → allowed: that arc is the chapter working. Confirm the token store holds only **hashes**, and that re-submitting a used token fails.

> 💡 **Hint — don't let verification leak existence either.** The resend and verify responses can re-introduce the enumeration problem from Chapter 20 ("we sent a link" vs "no such account"). Keep them generic — *"if that account exists and is unverified, a link has been sent"* — so verification endpoints don't become an email-confirmation oracle.

> **📖 Mandatory read — before Chapter 24.** Read about **secure email-verification token design** (search *"email verification token best practices single use expiry"*). *Required: the single-use + expiry + hashed pattern is the same one your refresh and (later) password-reset flows rely on.*

> **Interesting to read.** Many real breaches and spam waves trace back to *missing or weak* verification — services overwhelmed by fake accounts, or verification links that were guessable or never expired. Search *"account verification bypass"* to see how often the weak link is the token design, not the idea.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 24:

- [ ] `accounts.email_verified` exists (new migration), defaulting to **false**
- [ ] Registration issues a **single-use, expiring, hashed** verification token and (in dev) logs the link
- [ ] `POST /auth/verify` flips the account to verified and **consumes** the token; a used/expired token is rejected
- [ ] A **resend** endpoint issues a fresh token and invalidates the old
- [ ] At least one real action (publish a product / open a store) **requires verification** and returns **`403`** otherwise — you saw blocked → verify → allowed
- [ ] The token store holds **only hashes**; verification responses don't leak account existence

> **✍️ Log it (mandatory).** In `learning-log/23-email-verification.md` — **decision** first, then **topics**: **(Decision)** Why must an account start *unverified*, and what's the right line for which actions require verification in this marketplace? Why is the email send stubbed now instead of built fully? **(Topics)** (1) Why must the verification token be single-use, expiring, *and* stored hashed — what does each property defend? (2) How does verification reduce fake/spam vendor accounts? (3) Why return `403` (not `401`) when a *logged-in but unverified* user tries a gated action? (4) How could a careless verify/resend endpoint re-introduce account enumeration?

*All boxes ticked and the log written? Then continue. You can issue tokens and verify emails — but nothing yet *checks* the access token on incoming requests. Time to build the gate that guards every protected route.*

---

Next: tokens are issued but unenforced — build the one middleware that verifies them on every protected request and finally answers "who is calling?" → **[Chapter 24 — Authentication middleware](24-auth-middleware.md)**
