# Chapter 21 — Access tokens

Login works (Chapter 20) — you verify the password and say "yes, that's you." And then the response leaves, and your server *forgets you completely*. The very next request arrives an anonymous stranger again, because **HTTP is stateless**: every request is independent, carrying no memory of the last. So the real question of authentication isn't "is this password right?" — you solved that — it's **"how does the server remember who you are across requests?"**

This chapter answers it, and there are two genuinely different answers worth understanding before you pick one. Then you'll issue your first **access token** on login. (Making that token actually *guard* your routes is Chapter 24 — here you build and understand the token itself.)

## Where we're headed

By the end, logging in returns a signed **JWT access token** that carries the user's identity, your app knows what a JWT is and why its signature matters, you've made a deliberate decision about **where the client stores it**, and you understand why it's deliberately **short-lived** (the cliffhanger Chapter 22 resolves).

## Step 1 — The decision: how the server remembers you

Two real approaches, and the mentee should understand both before choosing.

**Stateful sessions.** On login, the server creates a **session record** (server-side) and hands the client a meaningless **session id**. Each request sends the id; the server looks it up to find who you are. The *truth lives on the server*.
- *Pro:* logging out is trivial — delete the session, and the id is instantly worthless. Full server control.
- *Con:* every server in a fleet must reach the **same session store** (a shared Redis/DB), and that lookup happens on *every* request.

**Stateless tokens (JWT).** On login, the server hands the client a **signed token that carries the claims** (who you are, your role). Each request sends the token; the server **verifies the signature** and reads the claims — *no lookup*. The truth travels *with the request*.
- *Pro:* any server can verify any request with no shared store — perfect for a scalable, multi-instance API.
- *Con:* "log out everywhere, right now" is genuinely hard, because there's nothing central to delete — a valid token is valid until it expires.

| | Stateful sessions | Stateless tokens (JWT) |
|---|---|---|
| Where truth lives | Server store | In the token |
| Per-request cost | A store lookup | A signature check (no lookup) |
| Scaling | Needs shared session store | Any instance verifies independently |
| Instant logout | Easy | Hard (mitigated by short expiry + refresh) |

**The course uses stateless JWT access tokens.** They fit an API meant to scale across instances without a shared session store, and the one real downside — hard revocation — you'll tame in Chapter 22 with short expiry plus revocable refresh tokens. (This is the classic stateless-vs-stateful trade-off — worth reading a short "sessions vs JWT" piece right here, because it only clicks once you've met both sides.)

## Step 2 — What a JWT actually is

Don't treat the token as a magic string — understand it, because a misunderstanding here is a security hole. A **JWT** (JSON Web Token) is three base64url parts joined by dots:

```
header . payload . signature
eyJhbG… . eyJzdW… . 4f9a…
  │          │          └─ signature: proves the token wasn't tampered with
  │          └─ payload: the CLAIMS — e.g. { "sub": "<account id>", "role": "vendor", "exp": ... }
  └─ header: the algorithm used
```

Two facts you must internalise:

- **It is signed, not encrypted.** The payload is *base64, not secret* — anyone holding the token can decode and read the claims. So **never put anything sensitive in it** (no password, no private data). It's a readable badge, not a sealed envelope.
- **The signature is what makes it trustworthy.** The server creates the signature using your secret **`JWT_SECRET`** (which you already validated into config back in Chapter 6). Change a single character of the payload — say, flip `"role": "customer"` to `"role": "admin"` — and the signature no longer matches, so verification fails. That's the whole security model: anyone can *read* a JWT, but only the holder of `JWT_SECRET` can *forge a valid one*. Keep that secret out of the client, out of Git, and strong (Chapter 6).

## Step 3 — Issue an access token on login

Now upgrade Chapter 20's login. On a successful password check, **sign a JWT** carrying the user's identity — their account id (`sub`) and role — with a **short expiry** (Step 5), using `JWT_SECRET`, and return it:

```
POST /auth/login  (success)
  → 200  { "accessToken": "eyJhbGc...", "user": { "id": "...", "role": "vendor" } }
```

The client stores this token and sends it on subsequent requests (in an `Authorization: Bearer <token>` header). The role lives *in the token* now — which is why a route can later check "are you a vendor?" without a database lookup. (The verifying gate that reads this on every protected request is Chapter 24.)

## Step 4 — Where does the client keep it? (the decision from Chapter 18)

Chapter 18 flagged this as a real security decision; now make it. The token has to live somewhere on the client, and the options carry different risks:

- **`localStorage`** — simple, but readable by *any* JavaScript on the page, so a single **XSS** flaw leaks every token. Persistent and exposed.
- **In memory (a JS variable)** — not persisted, so it's gone on refresh, but far less exposed to XSS than `localStorage`. Paired with a refresh token (Chapter 22) to survive reloads.
- **`httpOnly` cookie** — unreadable by JavaScript (XSS can't steal it), but brings **CSRF** considerations.

| Storage | XSS exposure | Survives refresh | Notes |
|---|---|---|---|
| `localStorage` | High (any script reads it) | Yes | Convenient, riskiest |
| In memory | Low | No | Needs refresh token to persist |
| `httpOnly` cookie | None to JS | Yes | Must handle CSRF |

**The course's approach: keep the short-lived *access* token in memory** (sent as a `Bearer` header), and in Chapter 22 keep the long-lived *refresh* token in an **`httpOnly` cookie**. That combination minimises XSS exposure of the powerful long-lived credential while keeping the access token easy to attach to API calls. Pick this and stay consistent.

## Step 5 — Why short-lived (and the cliffhanger)

Make the access token **short-lived** — minutes, not days (15 minutes is common). The reason is straight from the threat model: **a stolen token is an impersonation, so limit its blast radius in time.** Because a JWT is hard to revoke (Step 1), expiry *is* your revocation: a leaked token stops working on its own, fast.

But short expiry has an obvious cost — if the token dies every 15 minutes, is the user logged out mid-checkout, forced to re-enter their password constantly? That tension is real, and solving it is exactly what **refresh tokens** (Chapter 22) are for. For now, accept the short expiry as correct and feel the gap it opens.

## Step 6 — Do it on your project (hands-on)

**1. Issue a token on login.** Update `POST /auth/login` to sign a JWT (account id + role, short `exp`, signed with `JWT_SECRET`) and return it. Install a maintained JWT library — don't hand-build signing.

**2. Inspect what you issued.** Copy the returned token and decode it (paste into the debugger at `jwt.io`, or base64-decode the middle part) and *see* your claims:

```bash
curl -s -X POST localhost:3000/auth/login -d '{"email":"m@x.com","password":"a-long-passphrase"}' -H 'content-type: application/json'
# → { "accessToken": "eyJ...", ... }   then decode the middle segment → { "sub":"<id>", "role":"vendor", "exp":... }
```

**3. Prove the signature protects it.** Take the token, change one character in the payload segment, and verify it with `JWT_SECRET` (a tiny script using your JWT library): verification **fails**. That's tamper-detection — the claims can't be forged without the secret.

**4. Confirm it's readable, not secret.** Notice you could *read* the claims without the secret. Internalise the rule: nothing sensitive goes in the payload.

> 💡 **Hint — `JWT_SECRET` is the whole castle.** Every account's security rests on that one secret staying secret. If it leaks, an attacker can forge a valid token for *any* user — including `role: "admin"`. So it stays in env/config (Chapter 6), never in the client bundle, never in Git, and if it ever leaks you **rotate** it (which invalidates all existing tokens — a feature, not a bug). Treat it like the master key it is.

> **📖 Mandatory read — before Chapter 22.** Read an **"introduction to JWT"** (search *"jwt.io introduction"*) and a short **"sessions vs JWT"** comparison. *Required: Chapter 22 builds refresh tokens directly on this model, and the trade-offs only make sense once you've met both approaches.*

> **Interesting to read.** A recurring real-world JWT mistake is the **`alg: none` / algorithm-confusion** attack, where servers that don't pin their verification algorithm could be tricked into accepting *unsigned* or weakly-signed tokens. Search *"JWT alg none attack"* — it's a vivid reminder that a JWT is only as safe as the verification around it (which is why you use a vetted library and verify strictly in Chapter 24).

## Definition of Done

Things you can **see or run** — and the gate to Chapter 22:

- [ ] `POST /auth/login` returns a **signed JWT access token** carrying the account id and role, signed with `JWT_SECRET`
- [ ] You **decoded** the token and saw its claims — and understand the payload is **readable, not encrypted**
- [ ] You **tampered** with the payload and confirmed verification **fails** — the signature detects it
- [ ] The token is **short-lived** (minutes), and you can explain why expiry stands in for revocation
- [ ] You've made and recorded a **token-storage decision** (access token in memory / `Bearer` header), with the XSS/CSRF reasoning
- [ ] Nothing sensitive is placed in the token payload

> **✍️ Log it (mandatory).** In `learning-log/21-access-tokens.md` — **decision** first, then **topics**: **(Decision)** Why did this project choose stateless JWTs over stateful sessions — what does each trade away? Where will the client store the access token, and why that over the alternatives? **(Topics)** (1) Why is a JWT *signed but not encrypted*, and what must therefore never go in its payload? (2) What exactly does the signature (made with `JWT_SECRET`) prevent? (3) Why is the access token deliberately short-lived, and what problem does that create? (4) What happens to all existing tokens if you rotate `JWT_SECRET`, and why is that useful?

*All boxes ticked and the log written? Then continue. Your token expires in minutes — which is safe but would log users out constantly. Next you fix that without weakening anything.*

---

Next: a 15-minute token is safe but punishing — give users a way to stay logged in that you can also *revoke*. → **[Chapter 22 — Refresh tokens](22-refresh-tokens.md)**
