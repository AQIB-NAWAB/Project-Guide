# Chapter 59 — Security hardening

You've built security in *throughout* — hashing (19), auth (20–24), isolation (27–30), rate limiting (56). This chapter is the final sweep: the layer of production-grade defaults that a default-configured app is missing, and the last few rows on your Chapter 18 threat model. None of it is glamorous — security headers, CORS, body limits, dependency audits — but each one closes a real, common hole, and shipping without them is how otherwise-solid apps get embarrassed. Think of it as the pre-flight checklist before the app meets the open internet.

## Where we're headed

By the end your app sends sensible **security headers**, has **CORS** locked to known origins, enforces **body-size limits**, leaks **no internals** in production, uses **secure cookies**, passes a **dependency audit**, and your threat model has no open rows.

## Step 1 — Security headers

By default, your responses are missing headers that browsers use to defend users. A header middleware (the standard one for your stack — e.g. `helmet` for Express/Fastify) sets them all:

- **HSTS** (`Strict-Transport-Security`) — tells browsers to only ever use HTTPS for your domain (pairs with the TLS you add in Chapter 60).
- **`X-Content-Type-Options: nosniff`** — stops browsers guessing content types (a defence against some upload attacks — relevant given Chapter 43).
- **`X-Frame-Options` / frame-ancestors** — stops your site being embedded in a malicious iframe (clickjacking).
- **Content-Security-Policy** — controls what scripts/resources can load; the strongest defence against **XSS** (the threat behind your token-storage decision, Chapter 21). A real CSP takes tuning; even a basic one helps.

One middleware, a big jump in baseline safety.

## Step 2 — CORS: lock it to your origins

**CORS** (cross-origin resource sharing) controls which *websites* may call your API from a browser. The dangerous default is "allow everyone" (`*`) — which lets any site make authenticated requests on your users' behalf. Configure CORS to allow **only your known frontend origin(s)** (your dashboard, Chapter 34; your storefront), with credentials enabled only for those. This is especially important because your refresh token is a cookie (Chapter 22) — a permissive CORS policy combined with cookies is a CSRF risk. Set the allowed origins from config (Chapter 6), so dev and prod differ correctly.

## Step 3 — Limits and leaks

A few more deliberate defaults:

- **Body-size limits** — cap request body size, so a giant payload can't exhaust memory (a cheap DoS). You already limit *file* uploads at the presign step (Chapter 43); this caps *JSON* bodies.
- **No internals in production** — confirm your error handler (Chapter 17) returns generic `500`s with no stack traces in production (the leak you guarded against in Chapters 7 and 17), and that you've disabled framework fingerprinting headers (e.g. `X-Powered-By`) that advertise your stack.
- **Secure cookies** — your refresh-token cookie (Chapter 22) should be `httpOnly`, `Secure` (HTTPS-only), and `SameSite` — verify all three in production config.

## Step 4 — Audit your dependencies

Most of your code is *other people's* code (your dependencies), and they have vulnerabilities discovered constantly. Run a **dependency audit** (`npm audit`) to find known-vulnerable packages, and make it a habit (and later, a CI step). A critical vuln in a transitive dependency is as real as a bug in your own code — supply-chain issues are a major modern threat. Update or replace flagged packages.

## Step 5 — Close the threat model

This chapter is also your moment to walk `THREAT-MODEL.md` (Chapter 18) top to bottom and confirm **every row is addressed**:

| Threat | Mitigation | Chapter |
|---|---|---|
| Credential DB theft | Hashing + salt + slow hash | 19 |
| Brute force / stuffing | Rate limiting + slow hash | 56, 19 |
| Interception (MITM) | HTTPS/TLS | 60 (next) |
| Token theft | Short-lived + revocable tokens | 21–22 |
| Account enumeration | Uniform responses | 20 |
| Fake accounts | Email verification | 23 |
| IDOR / object access | Isolation + ownership checks | 27–30 |
| Function-level authz | Role checks | 26 |
| XSS | CSP + token-in-memory | 59, 21 |
| CSRF | SameSite cookies + CORS | 22, 59 |
| Injection | Validation + parameterized queries | 16, 33 |

Every row should now point at a chapter you've completed (TLS lands next, in deploy). If a row is still open, *that's* your remaining work.

## Step 6 — Do it on your project (hands-on)

**1. Add a security-headers middleware** (helmet-equivalent) and verify the headers on a response.

**2. Configure CORS** to your known origins from config; confirm a disallowed origin is rejected.

**3. Set a body-size limit**; confirm an oversized JSON body is rejected.

**4. Verify no leaks** — production error responses carry no stack traces; `X-Powered-By` is off; cookies are `httpOnly`/`Secure`/`SameSite`.

**5. Run `npm audit`** and address findings.

**6. Walk the threat model** and confirm every row is closed (TLS pending Chapter 60).

```bash
# headers present
curl -sI localhost:3000/products | grep -iE 'strict-transport|x-content-type|x-frame|content-security'

# CORS rejects an unknown origin
curl -sI -H "Origin: https://evil.example" localhost:3000/products | grep -i access-control-allow-origin   # → not the evil origin

# oversized body rejected
curl -i -X POST localhost:3000/products -H 'content-type: application/json' --data-binary @huge.json        # → 413 Payload Too Large

# dependency audit
npm audit
```

Headers present, unknown origins rejected, oversized bodies refused, and a clean audit — that's a hardened app. With the threat model fully closed (bar TLS), you're ready to ship.

> 💡 **Hint — security is layered, and never "done."** No single item here is *the* defence — they stack with everything you built (a hardened header doesn't replace an ownership check). And hardening isn't a one-time task: new dependency vulns appear, new headers become best practice. Bake `npm audit` into CI, keep dependencies current, and revisit this checklist periodically. "We added helmet once in 2026" is not a security posture.

> **📖 Mandatory read — before Chapter 60.** Read the **OWASP Top 10** end to end now that you've defended against most of it, and your security-header middleware's docs (search *"helmet security headers"*). *Required: a final cross-check before exposing the app publicly.*

> **Interesting to read.** You can score your live site's headers and config with free tools (Mozilla Observatory, Security Headers) that grade exactly the things in this chapter — a quick, humbling reality check. Search *"securityheaders.com"* and run your deployed app through it after Chapter 60.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 60:

- [ ] **Security headers** are set (HSTS, nosniff, frame options, a CSP); you verified them on a response
- [ ] **CORS** is locked to known origins (from config); an unknown origin is rejected
- [ ] **Body-size limits** reject oversized payloads; **file** limits remain (Chapter 43)
- [ ] Production leaks **nothing** — no stack traces, no `X-Powered-By`; cookies are `httpOnly`/`Secure`/`SameSite`
- [ ] **`npm audit`** is clean (or findings triaged), and you'll keep it in CI
- [ ] Your **threat model has no open rows** (except TLS, landing in Chapter 60)

> **✍️ Log it (mandatory).** In `learning-log/59-security-hardening.md` — **decision** first, then **topics**: **(Decision)** Why is hardening a *layered* set of defaults rather than one fix, and why must `npm audit` be ongoing, not one-time? **(Topics)** (1) What do HSTS, nosniff, frame options, and CSP each defend against? (2) Why is a wildcard CORS policy dangerous, especially with cookie-based auth? (3) Why do body-size limits and disabling `X-Powered-By` matter? (4) Walk your threat model and name which chapter closes each row.

*All boxes ticked and the log written? Then continue. The app is built, tested, observable, and hardened — there's only one thing left. Ship it.*

---

Next: everything is ready — deploy the marketplace to the real internet, over HTTPS, with secrets in the environment. → **[Chapter 60 — Deploy](60-deploy.md)**
