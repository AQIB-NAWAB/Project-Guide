# Chapter 25 — Authorization: who can do what

Your app now knows *who* is calling every protected route (Chapter 24) — `req.user` carries a verified identity. But knowing who someone is and knowing what they're *allowed to do* are different questions, and right now your app answers only the first. A logged-in **customer** can still call `POST /products` — they're authenticated, so `requireAuth` waves them through, even though creating products is a *vendor's* job. And a logged-in **vendor** could still try to edit *another* vendor's product by guessing its id. Authentication opened the door; **authorization** decides which rooms you may enter.

This is a concept chapter — the map for the authorization work ahead (this chapter and the whole isolation module, Chapters 26–30). You'll learn the two distinct kinds of authorization, the two ways they fail, and the principles that govern them — and you'll produce an **authorization matrix** for your own marketplace that the next chapters implement.

## What you'll be able to explain

By the end you can distinguish authentication from authorization cleanly; explain **role-based** vs **ownership-based** authorization; name the two failure modes (**function-level** and **object-level / IDOR**); apply **deny-by-default** and **least privilege**; and explain why a user's permissions must come from the server, never the request.

## Section 1 — Authentication vs authorization, sharpened

You met this distinction in Chapter 18; now it becomes the thing you build. **Authentication** answered *"who are you?"* and gave you `req.user`. **Authorization** asks *"is this user allowed to do **this**, to **this thing**?"* — and it always runs *after* authentication, because you can't decide what someone may do until you know who they are.

A perfectly authenticated user can be unauthorized in two different ways, and the difference matters:

- They're the wrong *kind* of user for this action (a customer trying to create products).
- They're the right kind, but this particular *object* isn't theirs (a vendor editing another vendor's product).

Those two are **function-level** and **object-level** authorization, and they need different defences.

## Section 2 — Two models: roles and ownership

**Role-based authorization (RBAC).** Permissions attach to a **role**. A vendor may create products; a customer may place orders; an admin may do administrative things. The check is "does this user's role permit this *action*?" — it doesn't care *which* product, just whether vendors-in-general may create products. This guards the **function**. (You'll build it next chapter.)

**Ownership-based authorization.** Permission depends on the **relationship between the user and the specific object**. A vendor may edit *their own* product, not just any product. The check is "does this user *own* this *particular* thing?" — and it's where your whole Chapter 8 ownership chain (product → store → vendor) finally gets used in anger. This guards the **object**, and at marketplace scale it's called **multi-tenant isolation** — keeping every vendor's data sealed off from every other's (the entire next module, Chapters 27–30).

Most real actions need *both*: to edit a product you must be a **vendor** (role) *and* its **owner** (ownership). Roles get you into the building; ownership gets you into your own office.

## Section 3 — The two failure modes (and why both are common breaches)

Each model has a matching failure, and both sit near the top of the OWASP API Security risks:

- **Broken function-level authorization** — a route that checks *authentication* but not *role*, so any logged-in user can hit an endpoint meant for one role. Your current `POST /products` is exactly this: a customer's valid token gets in. Chapter 26 fixes it.
- **Broken object-level authorization (IDOR)** — a route that checks role but not *ownership*, so a vendor edits `/products/<someone-else's-id>` simply by changing the id. This is the **IDOR** attack you designed *against* with opaque UUIDs back in Chapter 8 — but opaque ids only make ids hard to *guess*; they don't *enforce* ownership. That enforcement is Chapters 28–29.

The lesson: authentication alone is not access control. A route is only safe once it has checked *both* "may this role do this?" *and* "does this user own this object?" — wherever each applies.

## Section 4 — The governing principles

Four rules steer every authorization decision you'll write:

- **Deny by default.** A route is protected unless explicitly made public. New endpoints should be *closed* until you decide who may use them — the opposite of "open until someone notices." Forgetting to *add* a check should fail safe (no access), not fail open.
- **Least privilege.** Give each role exactly the permissions it needs, no more. A customer doesn't need product-write; a vendor doesn't need to see other vendors' orders.
- **Authorize on the server, from trusted data.** A user's role and ownership come from the **verified token and the database**, *never* from the request body. A request claiming `{"role":"admin"}` means nothing — that's the mass-assignment lesson (Chapter 15) wearing an authorization hat. The client states *intent*; the server decides *permission*.
- **Check at the right layer, every time.** Function checks guard the route; ownership checks guard the specific object, usually right where you load it. A missing check on one route is the whole hole — consistency is the defence.

## Section 5 — Map authorization for your marketplace (hands-on)

Make this concrete by writing the rules down for *your* app, before you implement them.

**1. Build an authorization matrix.** In `docs/AUTHORIZATION.md`, list your key actions against the actors, and mark what's allowed — and crucially, *which kind* of check each needs:

```
Action                     | anon | customer | vendor            | needs
---------------------------|------|----------|-------------------|---------------------------
Browse published products  |  ✓   |    ✓     |        ✓          | none (public)
Create a product           |  ✗   |    ✗     |        ✓          | role: vendor
Edit / delete a product    |  ✗   |    ✗     |  ✓ if they own it | role: vendor + OWNERSHIP
Place an order             |  ✗   |    ✓     |        ✓          | role: customer
View an order              |  ✗   | own only | own store's only  | OWNERSHIP
```

**2. Audit the current gaps.** Against that matrix, note where your app stands today: `POST /products` checks auth but **not role** (a customer gets in — a function-level hole); nothing checks **ownership** on any per-object route yet (an IDOR hole waiting). These are the exact gaps Chapter 26 and the isolation module close.

**3. Tick your threat model.** Return to `THREAT-MODEL.md` (Chapter 18): the "acting beyond your role / IDOR" row is what this module addresses — link it to your new matrix.

Commit both. The matrix is the spec you'll implement next; the audit is your proof you understand the holes before you patch them.

> **📖 Mandatory read — before Chapter 26.** Read the **OWASP API Security Top 10** entries on **Broken Object Level Authorization (BOLA/IDOR)** and **Broken Function Level Authorization** (search those names). *Required: these two are *the* most common API authorization failures, and the next chapters defend against exactly them.*

> **Interesting to read.** IDOR / broken object-level authorization is consistently ranked the **#1 API security risk**, and real companies have leaked huge volumes of customer data because an endpoint checked *that you were logged in* but never *that the record was yours*. Search *"BOLA IDOR API breach"* to see how a missing four-line ownership check becomes a headline.

## Key takeaways

You can now explain:

- **Authorization runs after authentication** and answers "may this user do *this*, to *this object*?" — being logged in is not being allowed.
- **Role-based** authorization guards the *action* (function-level); **ownership-based** authorization guards the *specific object* (object-level / multi-tenant isolation), and most actions need both.
- The two failure modes — **broken function-level** auth and **broken object-level (IDOR)** — are among the most common real breaches, and authentication alone prevents neither.
- The governing principles: **deny by default**, **least privilege**, and **authorize from server-trusted data, never the request body**.

> **✍️ Log it (mandatory) — this is the gate.** In `learning-log/25-authz-concept.md` — **decision** first, then **topics**: **(Decision)** For *your* marketplace, give one action that needs a *role* check, one that needs an *ownership* check, and one that needs *both*, and justify each. Why must a user's role come from the verified token and never the request? **(Topics)** (1) Explain function-level vs object-level authorization with a concrete endpoint for each. (2) Why doesn't using opaque UUIDs (Chapter 8) make an ownership check unnecessary? (3) What does "deny by default" mean, and how does it make *forgetting* a check fail safe? (4) From your matrix: which current route has a function-level hole, and which has an ownership hole?

*Answered all of it, and committed your `AUTHORIZATION.md` matrix? Then continue. You know the rules — now enforce the first kind: make each route demand the right *role*.*

---

Next: turn the matrix into enforcement — start with role checks, so a customer can no longer reach a vendor-only endpoint. → **[Chapter 26 — Roles and route protection](26-roles-and-route-protection.md)**
