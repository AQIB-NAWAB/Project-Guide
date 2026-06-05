# Chapter 22 — Refresh tokens

Your access token is deliberately short-lived (Chapter 21) — safe, because a stolen one expires fast. But you felt the cost immediately: make it expire in 15 minutes and the user is logged out mid-checkout, asked for their password again and again. You can't fix that by making the access token long-lived — that just re-opens the theft window. You need a *second* credential with a different job.

That's the **refresh token**: a longer-lived credential whose *only* purpose is to mint fresh access tokens when the old one dies. And here's the elegant part — adding it doesn't just fix the usability problem, it *also* gives you back the revocation power that stateless JWTs gave up. This chapter is a real step up: you'll build a two-token system with rotation and revocation, and the password lessons from Chapter 19 come straight back.

## Where we're headed

By the end you'll have a `refresh_tokens` store (new migration), a `POST /auth/refresh` that exchanges a valid refresh token for a new access token, and a `POST /auth/logout` that **revokes** it — with refresh tokens stored **hashed**, **rotated** on every use, and kept in an `httpOnly` cookie.

## Step 1 — The two-token model

The split of responsibilities is the whole idea:

- The **access token** (Chapter 21) is short-lived, sent on every API request, and proves identity *fast* with no lookup. If stolen, it dies in minutes.
- The **refresh token** is long-lived (days/weeks), sent *only* to one endpoint (`/auth/refresh`), and its single job is to obtain a new access token. It is never sent on ordinary requests.

```
login            → access token (15 min)  +  refresh token (e.g. 30 days)
…access expires…
POST /auth/refresh (sends refresh token)   → a fresh access token  (+ a rotated refresh token)
POST /auth/logout  (sends refresh token)   → refresh token revoked; future refresh fails
```

The user stays logged in for weeks without re-entering a password, yet every *access* token in flight is only valid for minutes. Usability and safety at once.

## Step 2 — The refresh token is the password problem again

A refresh token is powerful — anyone holding a valid one can mint access tokens, i.e. *be* that user for weeks. So a refresh token sitting **readable** in your database is a master key, and Chapter 19's lesson applies verbatim: **store it hashed, never in plaintext.** If your DB leaks, the stored hashes are useless to the attacker — they can't be presented as tokens.

So you store a *hash* of each refresh token (a fast hash like SHA-256 is acceptable here, since refresh tokens are already long and random — unlike low-entropy human passwords that need slow hashing). When a refresh token arrives, you hash it and look up the hash. You keep the token's metadata so you can manage it:

```
refresh_tokens
──────────────────────────────────────────────
 id          (uuid, primary key)
 account_id  (uuid, FK → accounts)
 token_hash  (text, not null)        ← the HASH, never the token itself
 expires_at  (timestamptz, not null)
 revoked_at  (timestamptz, null)     ← set on logout / rotation
 created_at  (timestamptz, not null)
```

This table is what makes a stateless system *revocable*: the access token still needs no lookup, but the *refresh* step checks this table — so revoking here cuts off the user's ability to renew, and within minutes (one access-token lifetime) they're effectively logged out everywhere. You got JWT's scalability *and* session-style revocation, just at different layers.

## Step 3 — Rotation: detect a stolen refresh token

Don't let a refresh token be reused forever. **Rotate** it: every time `/auth/refresh` is used, issue a *new* refresh token and **revoke the old one**. The old token becomes single-use.

This isn't just hygiene — it's a theft *detector*. If an attacker steals a refresh token and uses it, it rotates; when the *legitimate* user later presents the now-revoked old token, you see a **revoked token being reused** — a strong signal of theft. The standard response is to **revoke the entire chain** (log that user out everywhere and force re-login). Reuse detection is one of the main reasons rotation exists.

## Step 4 — Revocation and logout

Now logout is *real*. With stateless access tokens alone (Chapter 21), "log out" was almost meaningless — the token stayed valid until expiry. With the refresh table, `POST /auth/logout` sets `revoked_at` on the user's refresh token. The next `/auth/refresh` finds it revoked and returns `401`; the current access token still works for its last few minutes, then can't be renewed. That short tail is the price of stateless access tokens, and it's why "minutes, not hours" matters.

## Step 5 — Where the refresh token lives

The refresh token is your most powerful client-side credential, so protect it best. Per the Chapter 21 decision: store it in an **`httpOnly` cookie** — unreadable by JavaScript, so an XSS flaw can't steal it (unlike the access token, a refresh-token theft is catastrophic, so this one gets the strongest storage). Because cookies are sent automatically, add CSRF protection (a `SameSite` cookie attribute and/or a CSRF token) so a malicious site can't trigger a refresh on the user's behalf. Scope the cookie to the `/auth/refresh` path so it's only ever sent where it's needed.

## Step 6 — Do it on your project (hands-on)

**1. Migrate the store.** Add the `refresh_tokens` table with a new migration (Chapter 12), then `npm run migrate`.

**2. Issue both tokens on login.** Update `/auth/login` to also generate a refresh token, store its **hash** with an expiry, and set it as an `httpOnly` cookie. Return the access token as before.

**3. Build `POST /auth/refresh`.** It reads the refresh-token cookie, hashes it, looks it up, and checks it's **not expired and not revoked**. If valid: issue a new access token, **rotate** (revoke the old refresh row, create a new one, set the new cookie). If revoked-but-presented: treat as reuse — revoke the chain and `401`.

**4. Build `POST /auth/logout`.** Revoke the current refresh token (`revoked_at = now`) and clear the cookie.

**5. Prove the whole lifecycle:**

```bash
# login → get access token + refresh cookie (save cookies to a jar)
curl -i -c jar.txt -X POST localhost:3000/auth/login -d '{"email":"m@x.com","password":"a-long-passphrase"}' -H 'content-type: application/json'

# refresh → new access token, refresh cookie rotated
curl -i -b jar.txt -c jar.txt -X POST localhost:3000/auth/refresh        # → 200, new accessToken

# logout → revoke
curl -i -b jar.txt -X POST localhost:3000/auth/logout                    # → 200

# refresh again with the revoked token → rejected
curl -i -b jar.txt -X POST localhost:3000/auth/refresh                   # → 401  (revocation works!)
```

That final `401` is the proof that you have *real* logout — something stateless access tokens alone couldn't give you. And inspect the `refresh_tokens` table: you'll see only hashes and a `revoked_at` timestamp, never a usable token.

> 💡 **Hint — never return or log the refresh token's value after issuing it.** You store only its hash, so the *only* time the raw token exists is the moment you set the cookie. Don't log it, don't put it in a JSON body you also log, don't return it in an API response the frontend might cache. The hash in your DB is worthless to a thief; the raw value, briefly in flight, is not.

> **📖 Mandatory read — before Chapter 23.** Read about **refresh token rotation and reuse detection** (search *"refresh token rotation reuse detection"*) and **`SameSite` cookies / CSRF**. *Required: rotation and the cookie protections are the parts most often gotten wrong.*

> **Interesting to read.** Refresh-token *reuse detection* is how providers like Auth0 catch stolen sessions: the instant a already-rotated token reappears, they assume theft and kill the whole family of tokens. Search *"refresh token reuse detection automatic revocation"* to see the pattern you just built treated as a production security feature.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 23:

- [ ] A `refresh_tokens` table exists (new migration) storing **hashes**, expiry, and a `revoked_at`
- [ ] Login issues **both** a short-lived access token and a longer-lived refresh token (refresh in an **`httpOnly` cookie**)
- [ ] `POST /auth/refresh` exchanges a valid refresh token for a **new access token** and **rotates** the refresh token
- [ ] `POST /auth/logout` **revokes** the refresh token; a subsequent refresh returns **`401`** (you tested it)
- [ ] Presenting an already-rotated (revoked) refresh token is treated as **reuse** and revokes the chain
- [ ] The database stores **no usable refresh token** — only hashes

> **✍️ Log it (mandatory).** In `learning-log/22-refresh-tokens.md` — **decision** first, then **topics**: **(Decision)** Why solve the "short access token logs users out" problem with a *second* token rather than just making the access token long-lived? Why store refresh tokens hashed when they're already random? **(Topics)** (1) Explain the division of labour between access and refresh tokens. (2) How does the refresh-token table give a *stateless* system real revocation, and why is logout only "near-instant"? (3) What is token rotation, and how does it *detect* a stolen refresh token? (4) Why does the refresh token get an `httpOnly` cookie while the access token does not?

*All boxes ticked and the log written? Then continue. Login and sessions are solid — but you're still trusting that the email behind each account is real. Next you make people prove it.*

---

Next: anyone can register with an email they don't own — close that hole before a vendor opens a store under a stranger's address. → **[Chapter 23 — Email verification](23-email-verification.md)**
