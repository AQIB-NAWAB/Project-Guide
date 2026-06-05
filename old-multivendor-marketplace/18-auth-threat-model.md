# Chapter 18 — The authentication threat model

Week 1 gave you an API that speaks HTTP correctly — but it has a gaping hole it doesn't even know about: it has **no idea who is calling it.** Right now, anyone on the internet can `POST /products` and create a listing, because there is no notion of a *caller* at all. Every endpoint is wide open. Week 2 closes that hole by teaching your app *who* people are (authentication) and *what they're allowed to do* (authorization) — but before you write a single line of login code, you're going to do what security engineers do first: **model the threats.**

This is deliberate. Authentication is the one part of an app where a naive, "it works" implementation can be *completely broken* in ways you won't see until an attacker does. Building auth without a threat model is how you build auth that's worth nothing. So this concept chapter is the **map for the entire auth module** — you'll come out of it with a written threat model for your own marketplace that the next six chapters fill in, one defense at a time.

## What you'll be able to explain

By the end you can distinguish authentication from authorization; name the assets your marketplace must protect and the main threats against a login system; explain the **"assume breach"** mindset; and point to *which upcoming chapter* defends against each specific threat.

## Section 1 — Authentication vs authorization

Two questions people constantly blur, and the whole of Week 2 hangs on keeping them apart:

- **Authentication (authn) — *who are you?*** Proving identity. Logging in, holding a session, presenting a token. This module (Chapters 18–24) is authentication.
- **Authorization (authz) — *what are you allowed to do?*** Permissions. "A vendor may edit their *own* products, not someone else's." That's the *next* module (Chapters 25–30).

A concrete way to feel the difference: authentication is the bouncer checking your ID at the door; authorization is the usher deciding which seats your ticket lets you into. You can be perfectly *authenticated* (we know exactly who you are) and still *unauthorized* (you're not allowed to do this particular thing). This chapter, and the five after it, are about the bouncer. (Recall the three separate gates from Chapter 15 — validation, authentication, authorization; Week 2 builds the second and third.)

## Section 2 — Think like an attacker: assets, threats, mitigations

The wrong question is "how do I build a login form?" The right one is the security engineer's: **"What here is valuable, who wants it, how would they take it, and what stops them?"** That framing — assets → threats → mitigations — is threat modeling, and it changes what you build.

Start with the **assets** your marketplace must protect:

- Customer accounts and their personal data (addresses, order history).
- Vendor accounts and their **payout details** — directly money-adjacent.
- The integrity of *acting as* someone — placing orders, editing a store, as a real user.
- The credential database itself.

And notice something specific to *this* app: a marketplace has **multiple trust levels** — anonymous visitor, customer, vendor, (later) admin — which is a far larger attack surface than a single-user app. More roles means more ways to act as the wrong one.

## Section 3 — The threats, and where each is defended

Here's the heart of the chapter: the real threats against an auth system, the naive mistake each one punishes, and the future chapter that defends it. The naive mental model — *"auth means checking the submitted password equals the stored one"* — fails against nearly every row below.

| Threat | What goes wrong naively | Defended in |
|---|---|---|
| **Credential DB theft** | Passwords stored as plaintext (or reversibly "encrypted") → one leak exposes every account *and* every reused password elsewhere | Hashing + salt + slow hash — **Ch 19** |
| **Brute force / credential stuffing** | Unlimited fast guesses, or replaying leaked username/password lists | Slow hashes (Ch 19) + rate limiting (**Week 4**) |
| **Interception (MITM)** | Credentials/tokens sent over plain HTTP, sniffed in transit | HTTPS/TLS everywhere — **Ch 60** |
| **Token theft** | A stolen long-lived token = unlimited impersonation | Short-lived access tokens (**Ch 21**) + refresh rotation & revocation (**Ch 22**) |
| **Account enumeration** | Login/reset replies reveal whether an email exists ("no such user" vs "wrong password") → attacker harvests valid accounts | Uniform responses — **Ch 20** |
| **Fake / unverified accounts** | Signups with emails nobody owns; spam vendors | Email verification / OTP — **Ch 23** |
| **Acting beyond your role / IDOR** | A vendor reaches another's data by changing an id | Authorization & isolation — **Ch 25–30** |

This table *is* your roadmap for the module. Each upcoming chapter exists because of a row here.

## Section 4 — The through-line: "assume breach"

One mindset unifies every defense above: **assume each layer will eventually fail, and design so that failure isn't fatal.** Not *if* — *when*.

- The database *will* leak someday (a stray backup, an injection, an insider) → so store passwords such that they're **useless even when stolen** (Chapter 19). You don't hash passwords because you expect a leak tomorrow; you hash them because you're planning for the day one happens.
- A token *will* be stolen → so make it **short-lived and revocable** (Chapters 21–22), so a theft expires fast and can be cut off.
- The network *might* be tapped → so encrypt everything in transit (TLS).

This is **defense in depth** again (you met it with constraints in Chapter 11 and validation in Chapter 15): many independent layers, so no single failure hands over everything. Auth is where that principle matters most.

## Section 5 — A decision on the horizon: where tokens live

One threat deserves an early flag because it shapes a decision the token chapters will make: **where does the client store its token?** The two common choices each carry a risk:

- **`localStorage`** — easy, but readable by *any* JavaScript on the page, so a single **XSS** (cross-site scripting — injected malicious script) flaw leaks every user's token.
- **`httpOnly` cookies** — not readable by JavaScript (XSS can't steal them), but introduce **CSRF** (cross-site request forgery — a malicious site making requests with the user's cookie) concerns to handle.

Don't resolve it now — Chapters 21–22 will, with the trade-offs in front of you. Just record that "token storage" is part of your threat surface, and that XSS and CSRF are the two attacks pulling on it.

## Section 6 — Build your threat model (hands-on)

This is a concept chapter, but you'll still produce something real in your repo — the document that steers the whole module.

**1. Audit your current exposure.** Be honest about where the app stands *today*: it has **zero authentication**. Anyone can call `POST /products`; there is no caller identity, no protected routes, no concept of "your" data. Write down, plainly, what is currently open and who could abuse it — "any anonymous request can create a product in any store." Seeing your real, unguarded app through an attacker's eyes is the point.

**2. Write `docs/THREAT-MODEL.md`.** Create the artifact and fill it with:
- **Assets** — what's worth protecting in *your* marketplace (Section 2).
- **Trust levels** — anonymous, customer, vendor (and admin if you plan one).
- **Threats → mitigations → chapter** — adapt the Section 3 table to your app, so each threat names the chapter that will close it.

Commit it. As you build each defense over the next chapters, you'll tick its row — turning the threat model into a checklist you actually complete.

**3. Note the auth trust boundary.** Extend your Chapter 15 boundary map: credentials will arrive in a request body (so they're validated — Chapter 16), and tokens will arrive in a header (so they must be *verified* — Chapter 24). Auth doesn't bypass the boundary; it adds new, especially sensitive inputs to it.

> **📖 Mandatory read — before Chapter 19.** Read the **OWASP Authentication Cheat Sheet** (search that name) and skim the **OWASP Top 10** for the authentication- and access-control-related entries. *Required: these are the canonical, industry-standard distillations of exactly the threats above, and Chapter 19 starts implementing the first defense.*

> **Interesting to read.** In 2012, LinkedIn leaked ~6.5 million password hashes stored as **unsalted SHA-1** — a fast hash with no salt — and the vast majority were cracked within days. It's the single clearest real-world argument for everything Chapter 19 is about, and a perfect illustration of "assume breach": the leak was survivable *only* for the few who'd used strong, unique passwords. Search *"LinkedIn 2012 password breach"*.

## Key takeaways

You can now explain:

- **Authentication ("who are you?") vs authorization ("what may you do?")** — Week 2 builds both, in that order, and they are not the same gate.
- The **assets** a marketplace must protect and why its **multiple trust levels** widen the attack surface.
- The major **threats** to an auth system — credential theft, brute force, interception, token theft, enumeration, fake accounts — and the specific chapter that defends each.
- The **"assume breach"** mindset: design every layer to fail safely, because eventually one will.
- That **where a token is stored** is itself a security decision, with XSS and CSRF as the competing risks.

> **✍️ Log it (mandatory) — this is the gate.** In `learning-log/18-auth-threat-model.md` — **decision** first, then **topics**: **(Decision)** Why model threats *before* writing any auth code — what does building login "by feel" risk getting wrong? In your own marketplace, give one concrete example each of an *authentication* failure and an *authorization* failure. **(Topics)** (1) Explain the "assume breach" mindset and how it dictates the way passwords must be stored. (2) Describe the credential-database-theft threat and its mitigation. (3) Why is *account enumeration* a real threat, and what does a uniform response protect? (4) Why is a *short-lived, revocable* token safer than a long-lived one — what threat does that address?

*Answered all of it, and committed your `THREAT-MODEL.md`? Then continue. You know what you're defending and why — now build the first and most important defense: storing passwords so a database leak can't expose them.*

---

Next: the first defense on your threat model — store passwords so that even a full database leak hands an attacker nothing usable. → **[Chapter 19 — Password hashing](19-password-hashing.md)**
