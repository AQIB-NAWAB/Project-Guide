# Chapter 56 — Rate limiting

Your marketplace is feature-complete — but it's not yet safe to expose to the open internet, where not every caller is friendly. Right now, nothing stops someone from hammering your **login** endpoint with millions of password guesses (the brute-force / credential-stuffing threats from your Chapter 18 threat model), scraping your entire catalogue in seconds, or simply overwhelming your server with requests until it falls over. **Rate limiting** is the defence: cap how many requests a client can make in a window, and reject the excess. It's the first chapter of the production-hardening module (56–59), where you close the remaining rows on your threat model.

## Where we're headed

By the end, sensitive endpoints (login, register) have strict rate limits and the API has a sane global limit, all backed by Redis so limits hold across multiple instances, returning `429 Too Many Requests` with a `Retry-After` — and you've ticked the brute-force row on your threat model.

## Step 1 — What rate limiting defends against

Capping request rate per client defends several things at once:

- **Brute force / credential stuffing** (Chapter 18) — limiting login attempts turns "millions of guesses per minute" into "five attempts, then locked for a while," making password-guessing infeasible. This is the mitigation your threat model promised.
- **Scraping and abuse** — stops someone vacuuming your whole catalogue or spamming registrations.
- **Denial of service / overload** — a single client (malicious or a buggy script) can't consume all your capacity.
- **Cost control** — every request costs CPU, database, and (for jobs) downstream API calls; limits bound the damage.

## Step 2 — How it works, and the algorithms

A rate limiter tracks how many requests a **client** (by IP, or by user id for authenticated routes) has made in a time window, and rejects once they exceed the limit. The common algorithms, briefly:

- **Fixed window** — N requests per fixed clock window (e.g. per minute). Simple, but allows bursts at window edges.
- **Sliding window** — smooths the edges by considering a rolling window. More accurate.
- **Token bucket** — each client has a bucket of tokens that refills at a steady rate; each request spends one. Allows controlled bursts. A common, flexible choice.

You don't need to implement these from scratch — use a maintained rate-limit library — but know which behaviour you're getting. For login, you want strict and unforgiving; for general traffic, something that allows normal bursts.

## Step 3 — It must be shared across instances (Redis)

A critical subtlety: in production you run **multiple instances** of your API (Chapter 60). If each instance counts requests *in its own memory*, a client hitting different instances effectively multiplies their limit — and the limit is meaningless. So the counter must live in a **shared store**: **Redis** (already running, Chapter 5). A Redis-backed limiter means all instances share one count per client, so the limit holds no matter which instance handles each request. (This is the same "shared state across the fleet" reasoning as sessions in Chapter 21 — distributed systems need shared state for shared decisions.)

## Step 4 — Different limits for different endpoints

One global limit isn't enough — tune by sensitivity:

- **Login / register / password-related** — *strict* (e.g. a handful of attempts per IP/account per few minutes). These are the brute-force targets; combine with the account-level protections from auth (Chapter 20).
- **Expensive endpoints** (search, report generation) — moderate limits, since each costs more.
- **General API** — a generous global limit as a backstop against overload.

When a client exceeds the limit, return **`429 Too Many Requests`** (through your error contract, Chapter 17) with a **`Retry-After`** header telling them when to try again. Be careful not to leak information — a rate-limited login should still give the *uniform* response (Chapter 20), not reveal account existence.

## Step 5 — Do it on your project (hands-on)

**1. Add a Redis-backed rate-limit middleware** (a maintained library) so counts are shared across instances.

**2. Apply tiered limits** — strict on `POST /auth/login` and `/auth/register`, moderate on search, a generous global default. Return `429` + `Retry-After` via your error contract.

**3. Prove the limit bites (especially on login):**

```bash
# hammer login with wrong passwords — after N attempts, get rate limited
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:3000/auth/login \
    -d '{"email":"victim@x.com","password":"guess'$i'"}' -H 'content-type: application/json'
done
# → 401, 401, 401, ... then 429 (rate limited) with a Retry-After header
```

Watching login flip to `429` after a handful of attempts is the brute-force mitigation working — a guesser can't make millions of attempts. Confirm a `Retry-After` header is present and the response still doesn't leak account existence.

**4. Tick the threat model.** In `THREAT-MODEL.md` (Chapter 18), mark the **brute force / credential stuffing** row mitigated, referencing this chapter (plus the slow hash from Chapter 19).

> 💡 **Hint — identify the client carefully (IP isn't perfect).** Limiting by IP is the default, but many users share an IP (offices, mobile carriers, NAT), and attackers rotate IPs. For authenticated routes, also limit per *user/account*; for login, consider limiting per *account being targeted* (not just per IP) so an attacker can't bypass the limit by rotating IPs against one victim. And remember: behind a proxy/load balancer (Chapter 60), the real client IP comes from a forwarded header you must configure correctly — or everyone looks like the proxy.

> **📖 Mandatory read — before Chapter 57.** Read about **rate-limiting algorithms** (search *"rate limiting algorithms token bucket sliding window"*) and the **`429` / `Retry-After`** semantics. *Required: rate limiting is your primary brute-force defence and a core production concern.*

> **Interesting to read.** Big APIs publish their rate limits openly (GitHub, Twitter/X, Stripe) and return `429` with headers telling clients exactly how many requests remain — rate limiting is so fundamental it's part of the public API contract. Search *"github api rate limit headers"* to see how it's communicated at scale.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 57:

- [ ] A **Redis-backed** rate limiter caps requests, so limits hold **across instances**
- [ ] **Strict limits** on login/register; a **generous global** limit elsewhere
- [ ] Exceeding a limit returns **`429`** with a **`Retry-After`** header, via your error contract
- [ ] A rate-limited login **still gives the uniform response** (no enumeration leak)
- [ ] You **demonstrated** login getting rate-limited after repeated attempts
- [ ] The **brute-force row** on your threat model is marked mitigated

> **✍️ Log it (mandatory).** In `learning-log/56-rate-limiting.md` — **decision** first, then **topics**: **(Decision)** Why must the rate-limit counter live in Redis rather than each instance's memory? Why apply stricter limits to login than to general traffic? **(Topics)** (1) Which threats does rate limiting defend against (tie to Chapter 18)? (2) Briefly contrast fixed-window, sliding-window, and token-bucket. (3) Why is limiting by IP alone insufficient, and what else can you key on? (4) What status and header do you return, and why must login stay uniform?

*All boxes ticked and the log written? Then continue. Your app resists abuse — but when something *does* go wrong in production, can you see what happened? Next you make the app observable.*

---

Next: when something breaks in production at 3am, logs are all you have — make them structured and correlated. → **[Chapter 57 — Logging and request IDs](57-logging-and-request-ids.md)**
