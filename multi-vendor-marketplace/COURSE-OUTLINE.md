# FreshMarket — Multi-Vendor Grocery Marketplace (MERN)

**Course outline & chapter breakdown**

| | |
|---|---|
| **Stack** | MongoDB · Express · React (Vite) · Node.js · Redis · BullMQ · Cloudinary |
| **Domain** | Local grocery marketplace — vendors are local shops; products carry unit, weight, stock, perishability |
| **Difficulty** | Medium — assumes the mentee can already build CRUD |
| **Duration** | 4 weeks · ~60 hrs/week · progress by chapters finished, not clock |
| **Design** | Clean editorial style — name the product yourself |
| **Personalisation line** | *You're building for a local grocery market in a clean editorial style — name the product yourself.* |

**Phase demo targets (Rule 28 — every phase must be runnable):**

| Phase | Done when the mentee can demo… |
|---|---|
| Week 1 — Foundations | Both apps run; client hits `GET /health`; MongoDB connected; seed data visible in Compass |
| Week 2 — Vendor experience | Vendor registers, opens a store, adds a product with a photo, sees it in their dashboard |
| Week 3 — Buyer experience | Customer browses, adds items from two vendors to cart, checks out once, sees split orders |
| Week 4 — Polish & ship | Cached catalogue, search works, **async jobs after checkout**, app live on HTTPS |

---

## Course folder layout

```
multi-vendor-marketplace/               ← course content (this repo folder)
  COURSE-OUTLINE.md                     ← this file
  01-introduction/
  02-project-skeleton/
  03-why-mongodb/
  ...
  20-cache-and-search/
  21-async-jobs-and-queues/
  22-deploy/
  99-closing/
```

Chapters are numbered folders at the course root — **no phase subfolders**. Week groupings (Week 1–4) are labels in this outline, not directory names.

**Sub-chapter arc** (every feature chapter follows this skeleton):

| Cluster | ≈ count | Purpose |
|---|---|---|
| Open & motivate | 3 | Set the scene · why it matters · what you'll build |
| Learn it | 4–6 | Approaches · decision · deep-dive · mandatory reads |
| Check | 0–1 | Quiz placeholder (excluded for now) |
| Build it | 10–14 | Continuity recap · scaffold · one step per file/action · verify |
| Wrap | 2–3 | Practice (optional) · recap · checklist gate |

Concept chapters run shorter on *Build it* but still include applied repo work — never read-only.

**Beginner-friendly reading pattern:** Every build chapter’s `*.01-set-the-scene.md` includes **Bridge from last chapter**, **In plain English**, **Think of it like…**, and **Common beginner questions**. Build sub-chapters open with **Why this step matters**. Major week transitions use explicit bridge sub-chapters (e.g. `13.14` vendor → customer before Week 3).

---

## Chapter index

| # | Chapter folder | Phase | Done when… |
|---|---|---|---|
| 01 | `01-introduction/` | Mindset | Mentee understands the product, course rules, and checklist gate |
| 02 | `02-project-skeleton/` | Week 1 | `server/` + `client/` run; client displays health check response |
| 03 | `03-why-mongodb/` | Week 1 | Can explain why MongoDB fits this project and one case where SQL would win |
| 04 | `04-config-and-database/` | Week 1 | Config from env; MongoDB connects; `.env` not in Git |
| 05 | `05-data-model-and-seed/` | Week 1 | All schemas exist; ERD drawn; seed script populates test data |
| 06 | `06-authentication-api/` | Week 2 | Register/login; hashed passwords; JWT + refresh; OTP verification |
| 07 | `07-login-and-registration/` | Week 2 | Login/register screens work; token stored safely; refresh on 401 |
| 08 | `08-authorization-and-isolation/` | Week 2 | Role + ownership checks; vendor cannot touch another vendor's data |
| 09 | `09-vendor-store-api/` | Week 2 | Vendor creates/edits their store via API |
| 10 | `10-open-your-shop/` | Week 2 | Vendor onboarding UI; storeless vendor routed to open a shop |
| 11 | `11-manage-products-api/` | Week 2 | Vendor CRUDs own products; wrong-owner returns 403 |
| 12 | `12-vendor-dashboard/` | Week 2 | Vendor sees product table; create/edit/publish in UI |
| 13 | `13-product-photos/` | Week 2 | Photos in Cloudinary; URL only in MongoDB |
| 14 | `14-browse-catalogue-api/` | Week 3 | Public catalogue with filter, search, pagination; no N+1 |
| 15 | `15-shop-the-marketplace/` | Week 3 | Customer product grid with filters and pagination |
| 16 | `16-cart-api/` | Week 3 | Cart persists; multi-vendor items; user-scoped |
| 17 | `17-your-cart/` | Week 3 | Cart UI grouped by store; quantity controls |
| 18 | `18-checkout-and-orders/` | Week 3 | One checkout → one order per vendor; price snapshotted |
| 19 | `19-track-orders/` | Week 3 | Customer order history + vendor order list in UI |
| 20 | `20-cache-and-search/` | Week 4 | Redis cache with invalidation; text search on index |
| 21 | `21-async-jobs-and-queues/` | Week 4 | BullMQ jobs on Redis; checkout returns before notifications; worker process |
| 22 | `22-deploy/` | Week 4 | Live HTTPS URL; secrets in env; API + worker survive restart |
| 99 | `99-closing/` | Ship | Docs, demo video, case study, bar-raiser |

---

# Chapter breakdowns

---

## 01 — Introduction (`01-introduction/`)

*Mindset + the project. Shorter arc — no 20-step build cluster.*

| File | Cluster | What it covers |
|---|---|---|
| `01.01-what-youre-building.md` | Open | Why a multi-vendor marketplace is a career skill; one sentence on the finished app |
| `01.02-how-to-use-this-course.md` | Open | Chapters in order; checklist gate; learning log; how you're judged |
| `01.03-the-product-and-its-people.md` | Open | Product narrative first; then vendor & customer actors |
| `01.04-scope.md` | Open | In scope / out of scope; one-line user flow |
| `01.05-prerequisites.md` | Open | Prerequisites table pitched to medium level |
| `01.06-course-outline.md` | Open | This roadmap — phases, chapters, demo targets |
| `01.07-your-working-rhythm.md` | Wrap | Daily habits; commit; secrets; read errors first |

**Gate:** Mentee can explain what they're building, how progress is gated, and what's out of scope.

---

## 02 — Project skeleton (`02-project-skeleton/`)

*First build chapter. Nothing else starts until both apps run and talk.*

| File | Cluster | What it covers |
|---|---|---|
| `02.01-set-the-scene.md` | Open | What "standing the project on disk" means |
| `02.02-why-structure-matters.md` | Open | Layer vs module folders; real cost of picking wrong |
| `02.03-what-youll-build.md` | Open | Tree preview: `server/` + `client/` + `learning-log/` |
| `02.04-layer-vs-module.md` | Learn | Pros/cons table; **mandate module-based** for this course |
| `02.05-client-vs-server.md` | Learn | Browser vs API; HTTP; where secrets live — point at *their* folders |
| `02.06-prime-your-thinking.md` | Learn | Mandatory read: Node project layout basics |
| `02.07-quiz.md` | Check | Reserved — excluded |
| `02.08-create-the-repo.md` | Build | Init git; root `.gitignore`; `README.md` stub |
| `02.09-scaffold-server-folders.md` | Build | Create `server/src/modules/`, `server/src/shared/`, `server/src/app.js`, `server/src/server.js` |
| `02.10-server-package-and-scripts.md` | Build | `server/package.json`; express; dev script; verify `node` starts |
| `02.11-the-health-route.md` | Build | `GET /health` → `{ status: "ok" }`; curl → 200 |
| `02.12-scaffold-client-folders.md` | Build | Vite React app in `client/`; `src/pages/`, `src/components/`, `src/api/`, `src/hooks/` |
| `02.13-client-shell-and-routing.md` | Build | React Router; placeholder Home page; verify browser loads |
| `02.14-api-client-helper.md` | Build | `client/src/api/client.js` — base URL from env |
| `02.15-connect-client-to-server.md` | Build | Home page fetches `/health`; display JSON in UI |
| `02.16-verify-end-to-end.md` | Build | Both terminals running; screenshot-worthy first demo |
| `02.17-practice.md` | Wrap | Optional: add a `/api/version` route and show it in UI |
| `02.18-recap-and-whats-next.md` | Wrap | Tree diagram of repo as it stands now |
| `02.19-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Both apps run; folder tree matches course convention; client shows health response.

---

## 03 — Why MongoDB (`03-why-mongodb/`)

*Concept chapter — applied work on the real project, not essay-only.*

| File | Cluster | What it covers |
|---|---|---|
| `03.01-set-the-scene.md` | Open | Every marketplace needs a database — which shape fits? |
| `03.02-why-it-matters-for-grocery.md` | Open | Flexible product attributes (unit, weight, perishable) |
| `03.03-what-youll-decide.md` | Open | MongoDB for this course; when you'd pick SQL instead |
| `03.04-relational-vs-document.md` | Learn | Side-by-side; orders/ownership in both worlds |
| `03.05-the-decision-for-freshmarket.md` | Learn | **MongoDB + Mongoose** — mandated; why for this domain |
| `03.06-embedding-vs-referencing.md` | Learn | Order lines embedded; products reference stores |
| `03.07-prime-your-thinking.md` | Learn | Mandatory read: MongoDB data modelling introduction |
| `03.08-audit-your-planned-schema.md` | Build | Sketch ERD on paper; list collections you'll need |
| `03.09-map-actors-to-collections.md` | Build | User, Store, Product, CartItem, Order — one paragraph each |
| `03.10-checklist.md` | Wrap | Key takeaways + learning log questions |

**Gate:** Can explain why MongoDB fits; ERD sketch committed to repo (`docs/erd-draft.md`).

---

## 04 — Config & database (`04-config-and-database/`)

| File | Cluster | What it covers |
|---|---|---|
| `04.01-set-the-scene.md` | Open | Apps run but can't persist — time to connect MongoDB |
| `04.02-why-secrets-matter.md` | Open | LinkedIn/Git leak stories; `.env` discipline |
| `04.03-what-youll-build.md` | Open | Validated config; Mongoose connection; fail-fast on boot |
| `04.04-env-vs-hardcoded.md` | Learn | 12-factor config; `.env.example` vs `.env` |
| `04.05-local-mongodb.md` | Learn | Atlas vs local; pick one for course |
| `04.06-prime-your-thinking.md` | Learn | Mandatory read: Mongoose connection docs |
| `04.07-quiz.md` | Check | Reserved |
| `04.08-where-the-project-is-now.md` | Build | Recap Ch 02 tree |
| `04.09-create-env-files.md` | Build | `server/.env`, `.env.example`, `.gitignore` check |
| `04.10-config-module.md` | Build | `server/src/shared/config.js` — validate required vars at startup |
| `04.11-db-module.md` | Build | `server/src/shared/db.js` — Mongoose connect/disconnect |
| `04.12-wire-db-on-boot.md` | Build | Connect before `app.listen`; log success/failure |
| `04.13-health-check-db-status.md` | Build | Extend `/health` with `db: connected` |
| `04.14-client-env.md` | Build | `client/.env` — `VITE_API_URL`; update api client |
| `04.15-verify-connection.md` | Build | Stop MongoDB → app fails clearly; restart → recovers |
| `04.16-recap-and-whats-next.md` | Wrap | |
| `04.17-checklist.md` | Wrap | Definition of Done gate |

**Gate:** MongoDB connects from env; no secrets in Git; health shows DB status.

---

## 05 — Data model & seed (`05-data-model-and-seed/`)

| File | Cluster | What it covers |
|---|---|---|
| `05.01-set-the-scene.md` | Open | Schema is the most consequential decision |
| `05.02-the-denormalisation-trap.md` | Open | Copied `vendorName` on every product — explain, don't build wrong |
| `05.03-what-youll-build.md` | Open | All Mongoose models + seed script + ERD |
| `05.04-ownership-chain.md` | Learn | User → Store → Product → CartItem → Order |
| `05.05-grocery-domain-fields.md` | Learn | `unit`, `weightOrQuantity`, `isPerishable`, `stockQty` |
| `05.06-prime-your-thinking.md` | Learn | Mandatory read: Mongoose schemas & refs |
| `05.07-quiz.md` | Check | Reserved |
| `05.08-where-the-project-is-now.md` | Build | Recap: apps run, DB connected |
| `05.09-scaffold-models-folder.md` | Build | `server/src/modules/users/`, `stores/`, `products/`, `cart/`, `orders/` |
| `05.10-user-and-profile-models.md` | Build | Identity vs profile split — two schemas, linked |
| `05.11-store-model.md` | Build | Store belongs to vendor User |
| `05.12-product-model.md` | Build | Product belongs to Store; grocery fields |
| `05.13-cart-and-order-models.md` | Build | CartItem refs; Order embeds snapshot OrderItems |
| `05.14-seed-script-scaffold.md` | Build | `server/scripts/seed.js` — 2 vendors, 2 stores, products, 2 customers |
| `05.15-run-seed-and-inspect.md` | Build | Run seed; open Compass; read every document |
| `05.16-draw-final-erd.md` | Build | Commit `docs/erd.md` |
| `05.17-recap-and-whats-next.md` | Wrap | |
| `05.18-checklist.md` | Wrap | Definition of Done gate |

**Gate:** All schemas exist; seed runs; ERD in repo; mentee explains ownership chain.

---

## 06 — Authentication API (`06-authentication-api/`)

| File | Cluster | What it covers |
|---|---|---|
| `06.01-set-the-scene.md` | Open | Server treats every request as anonymous — fix that |
| `06.02-why-auth-matters.md` | Open | LinkedIn breach story; passwords and tokens |
| `06.03-what-youll-build.md` | Open | Register, login, refresh, OTP — API only |
| `06.04-sessions-vs-jwt.md` | Learn | Stateful vs stateless; pros/cons; **JWT for this course** |
| `06.05-password-hashing.md` | Learn | Hash + salt + bcrypt; never plain text |
| `06.06-what-a-jwt-is.md` | Learn | Header.payload.signature; role in payload |
| `06.07-prime-your-thinking.md` | Learn | Mandatory reads: bcrypt, JWT intro |
| `06.08-quiz.md` | Check | Reserved |
| `06.09-where-the-project-is-now.md` | Build | Models exist; no auth routes yet |
| `06.10-scaffold-auth-module.md` | Build | `auth.routes.js`, `auth.service.js`, `auth.middleware.js`, `auth.schema.js` |
| `06.11-wire-auth-routes.md` | Build | Mount `/api/auth` on app |
| `06.12-register-handler.md` | Build | `POST /auth/register` — hash password; role vendor/customer |
| `06.13-login-handler.md` | Build | `POST /auth/login` — compare hash; issue tokens |
| `06.14-access-token.md` | Build | Short-lived JWT with userId + role |
| `06.15-refresh-token.md` | Build | Hashed refresh in DB; `POST /auth/refresh`; revoke |
| `06.16-otp-verification.md` | Build | `POST /auth/verify-otp`; block unverified login |
| `06.17-auth-middleware.md` | Build | `requireAuth` — 401 on missing/invalid/expired |
| `06.18-test-with-curl.md` | Build | Full flow: register → verify → login → protected route |
| `06.19-recap-and-whats-next.md` | Wrap | Next: login screen |
| `06.20-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Auth API complete per plan competency map; no readable passwords in DB.

---

## 07 — Login & registration (`07-login-and-registration/`)

| File | Cluster | What it covers |
|---|---|---|
| `07.01-set-the-scene.md` | Open | API works; humans need screens |
| `07.02-why-token-storage-matters.md` | Open | XSS and refresh cookies |
| `07.03-what-youll-build.md` | Open | Register, login, OTP verify pages |
| `07.04-where-tokens-live.md` | Learn | Access in memory; refresh in httpOnly cookie |
| `07.05-the-401-retry-loop.md` | Learn | Refresh and retry once; else logout |
| `07.06-prime-your-thinking.md` | Learn | Mandatory read: JWT refresh on client |
| `07.07-quiz.md` | Check | Reserved |
| `07.08-where-the-project-is-now.md` | Build | Auth API tested via curl |
| `07.09-scaffold-auth-pages.md` | Build | `pages/Login.jsx`, `Register.jsx`, `VerifyOtp.jsx` |
| `07.10-auth-context-or-hook.md` | Build | `hooks/useAuth.js` or Context — access token in memory |
| `07.11-register-page.md` | Build | Form → `POST /auth/register`; role picker |
| `07.12-verify-otp-page.md` | Build | OTP input → verify endpoint |
| `07.13-login-page.md` | Build | Form → login; store token; redirect by role |
| `07.14-api-interceptor.md` | Build | Attach Bearer header; 401 → refresh → retry |
| `07.15-protected-route-wrapper.md` | Build | Redirect unauthenticated users to login |
| `07.16-verify-in-browser.md` | Build | Register vendor + customer in UI; both can log in |
| `07.17-recap-and-whats-next.md` | Wrap | |
| `07.18-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Login/register/verify work in browser; token refresh silent on 401.

---

## 08 — Authorization & isolation (`08-authorization-and-isolation/`)

| File | Cluster | What it covers |
|---|---|---|
| `08.01-set-the-scene.md` | Open | Logged in ≠ allowed |
| `08.02-idor-and-marketplaces.md` | Open | Vendor B edits Vendor A's store — real breach pattern |
| `08.03-what-youll-build.md` | Open | Role middleware + ownership checks |
| `08.04-authn-vs-authz.md` | Learn | Function-level vs object-level |
| `08.05-rbac-and-ownership.md` | Learn | Role check + store chain lookup |
| `08.06-prime-your-thinking.md` | Learn | Mandatory read: OWASP BOLA / IDOR |
| `08.07-quiz.md` | Check | Reserved |
| `08.08-authorization-matrix.md` | Build | Write `docs/AUTHORIZATION.md` — action × role × check type |
| `08.09-where-the-project-is-now.md` | Build | Auth works; no role guards yet |
| `08.10-require-role-middleware.md` | Build | `requireRole('vendor')` — 403 for wrong role |
| `08.11-ownership-helper.md` | Build | `assertStoreOwner(storeId, userId)` in shared |
| `08.12-apply-to-store-routes.md` | Build | Stub `PATCH /stores/:id` with both checks |
| `08.13-test-idor-with-two-vendors.md` | Build | Vendor B → 403 on Vendor A's store |
| `08.14-recap-and-whats-next.md` | Wrap | Next: vendor store feature |
| `08.15-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Role + ownership middleware exist; IDOR test returns 403; matrix in repo.

---

## 09 — Vendor store API (`09-vendor-store-api/`)

| File | Cluster | What it covers |
|---|---|---|
| `09.01-set-the-scene.md` | Open | Verified vendor needs a shop |
| `09.02-one-store-per-vendor.md` | Open | Duplicate store = confusing marketplace |
| `09.03-what-youll-build.md` | Open | Create + update store API |
| `09.04-store-fields.md` | Learn | Name, description, address, logoUrl |
| `09.05-prime-your-thinking.md` | Learn | Topics to read: slug vs id in URLs |
| `09.06-quiz.md` | Check | Reserved |
| `09.07-where-the-project-is-now.md` | Build | Auth + isolation middleware ready |
| `09.08-scaffold-stores-module.md` | Build | `stores.routes.js`, `stores.service.js`, `stores.schema.js` |
| `09.09-create-store-route.md` | Build | `POST /stores` — vendor only; one store enforced |
| `09.10-get-my-store-route.md` | Build | `GET /stores/me` |
| `09.11-update-store-route.md` | Build | `PATCH /stores/:id` — ownership check |
| `09.12-test-with-curl.md` | Build | Full vendor store lifecycle via API |
| `09.13-recap-and-whats-next.md` | Wrap | Next: onboarding UI |
| `09.14-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Vendor CRUDs own store via API; second store rejected; cross-vendor PATCH → 403.

---

## 10 — Open your shop (`10-open-your-shop/`)

| File | Cluster | What it covers |
|---|---|---|
| `10.01-set-the-scene.md` | Open | Vendor lands after login — no store yet |
| `10.02-onboarding-as-a-gate.md` | Open | Can't list products without a shop |
| `10.03-what-youll-build.md` | Open | Store setup form + routing gate |
| `10.04-prime-your-thinking.md` | Learn | Optional: multi-step forms UX |
| `10.05-where-the-project-is-now.md` | Build | Store API works |
| `10.06-scaffold-vendor-layout.md` | Build | `layouts/VendorLayout.jsx`, `pages/vendor/` |
| `10.07-store-setup-page.md` | Build | Form → `POST /stores` |
| `10.08-store-settings-page.md` | Build | Form → `PATCH /stores/:id` |
| `10.09-onboarding-redirect.md` | Build | No store → `/vendor/open-shop` before dashboard |
| `10.10-verify-onboarding-flow.md` | Build | New vendor forced through setup |
| `10.11-recap-and-whats-next.md` | Wrap | |
| `10.12-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Vendor opens shop in UI; storeless vendor cannot reach product pages.

---

## 11 — Manage products API (`11-manage-products-api/`)

| File | Cluster | What it covers |
|---|---|---|
| `11.01-set-the-scene.md` | Open | Empty shop — time to stock shelves |
| `11.02-two-views-of-products.md` | Open | Vendor inventory vs public catalogue |
| `11.03-what-youll-build.md` | Open | Vendor product CRUD API |
| `11.04-public-vs-vendor-routes.md` | Learn | `/vendor/products` vs `/products` |
| `11.05-prime-your-thinking.md` | Learn | Mandatory read: Mongoose populate |
| `11.06-quiz.md` | Check | Reserved |
| `11.07-where-the-project-is-now.md` | Build | Store exists for test vendor |
| `11.08-scaffold-products-module.md` | Build | routes, service, schema files |
| `11.09-create-product-route.md` | Build | `POST /products` — vendor + store ownership |
| `11.10-list-vendor-products.md` | Build | `GET /vendor/products` — scoped to auth vendor |
| `11.11-update-product-route.md` | Build | `PATCH /products/:id` — ownership |
| `11.12-delete-product-route.md` | Build | `DELETE /products/:id` — ownership |
| `11.13-test-cross-vendor-403.md` | Build | Vendor B cannot PATCH Vendor A's product |
| `11.14-recap-and-whats-next.md` | Wrap | |
| `11.15-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Vendor product CRUD works; isolation verified.

---

## 12 — Vendor dashboard (`12-vendor-dashboard/`)

| File | Cluster | What it covers |
|---|---|---|
| `12.01-set-the-scene.md` | Open | Vendor needs a back office |
| `12.02-table-plus-forms.md` | Open | Simple editorial table — enough for MVP |
| `12.03-what-youll-build.md` | Open | Product list, create, edit pages |
| `12.04-error-mapping.md` | Learn | 400 fields → inline errors; 403 → message |
| `12.05-where-the-project-is-now.md` | Build | Product API complete |
| `12.06-product-list-page.md` | Build | Table from `GET /vendor/products` |
| `12.07-create-product-page.md` | Build | Form with unit, weight, stock, perishable |
| `12.08-edit-product-page.md` | Build | Pre-fill + PATCH |
| `12.09-publish-toggle.md` | Build | Status field or isPublished toggle |
| `12.10-verify-vendor-flow.md` | Build | Create draft → edit → publish in UI |
| `12.11-recap-and-whats-next.md` | Wrap | |
| `12.12-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Vendor manages products entirely in browser.

---

## 13 — Product photos (`13-product-photos/`)

| File | Cluster | What it covers |
|---|---|---|
| `13.01-set-the-scene.md` | Open | Grocery needs photos; MongoDB is not a photo album |
| `13.02-base64-trap.md` | Open | Document bloat — explain cost, don't build wrong |
| `13.03-what-youll-build.md` | Open | Cloudinary upload; URL on product |
| `13.04-object-storage-pattern.md` | Learn | File → storage → URL in DB |
| `13.05-prime-your-thinking.md` | Learn | Cloudinary upload docs |
| `13.06-quiz.md` | Check | Reserved |
| `13.07-where-the-project-is-now.md` | Build | Products without images |
| `13.08-cloudinary-config.md` | Build | Env vars; server-side upload helper |
| `13.09-upload-route.md` | Build | `POST /products/:id/image` — multer → Cloudinary |
| `13.10-delete-old-image.md` | Build | Replace image; remove old from storage |
| `13.11-upload-ui-component.md` | Build | File input on create/edit product form |
| `13.12-verify-image-flow.md` | Build | Upload → URL in MongoDB → shows in list |
| `13.13-recap-and-whats-next.md` | Wrap | **Week 2 demo:** vendor shop with photos |
| `13.14-bridge-to-week-3-customer-experience.md` | Wrap | Role switch: vendor → customer; plain English; self-check before Week 3 |
| `13.15-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Images in Cloudinary only; product doc stays small.

---

## 14 — Browse catalogue API (`14-browse-catalogue-api/`)

| File | Cluster | What it covers |
|---|---|---|
| `14.01-set-the-scene.md` | Open | Customers need to see all vendors' products |
| `14.02-n-plus-one-trap.md` | Open | One query per product for store name — explain cost |
| `14.03-what-youll-build.md` | Open | Public `GET /products` with filters + pagination |
| `14.04-populate-vs-aggregate.md` | Learn | Mongoose `.populate('storeId')` |
| `14.05-pagination.md` | Learn | page + limit; default 20 |
| `14.06-prime-your-thinking.md` | Learn | Mandatory read: cursor vs offset pagination |
| `14.07-quiz.md` | Check | Reserved |
| `14.08-where-the-project-is-now.md` | Build | Vendor products exist; no public route |
| `14.09-public-products-route.md` | Build | `GET /products` — published, in-stock only |
| `14.10-filter-query-params.md` | Build | `?category=`, `?storeId=` |
| `14.11-pagination-params.md` | Build | `?page=`, `?limit=` |
| `14.12-populate-store-name.md` | Build | Store name without N+1 — verify query count |
| `14.13-single-product-route.md` | Build | `GET /products/:id` |
| `14.14-test-catalogue-api.md` | Build | curl with filters; log query count |
| `14.15-recap-and-whats-next.md` | Wrap | |
| `14.16-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Public catalogue API; N+1 fixed; pagination works.

---

## 15 — Shop the marketplace (`15-shop-the-marketplace/`)

| File | Cluster | What it covers |
|---|---|---|
| `15.01-set-the-scene.md` | Open | Customer-facing storefront |
| `15.02-editorial-grid.md` | Open | Clean cards — name, price/unit, store, photo |
| `15.03-what-youll-build.md` | Open | Browse page + product detail page |
| `15.04-where-the-project-is-now.md` | Build | Catalogue API ready |
| `15.05-scaffold-customer-pages.md` | Build | `pages/Shop.jsx`, `pages/ProductDetail.jsx` |
| `15.06-product-card-component.md` | Build | Reusable card |
| `15.07-filter-bar.md` | Build | Category + store filters |
| `15.08-shop-page-with-pagination.md` | Build | Grid + load more or page controls |
| `15.09-product-detail-page.md` | Build | Single product view + add to cart button stub |
| `15.10-verify-browse-ui.md` | Build | Browse across two vendors' products |
| `15.11-recap-and-whats-next.md` | Wrap | |
| `15.12-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Customer browses marketplace in browser with filters.

---

## 16 — Cart API (`16-cart-api/`)

| File | Cluster | What it covers |
|---|---|---|
| `16.01-set-the-scene.md` | Open | One cart, many vendors — split problem preview |
| `16.02-why-store-id-on-cart-item.md` | Open | Checkout must group by vendor |
| `16.03-what-youll-build.md` | Open | Persisted cart API |
| `16.04-prime-your-thinking.md` | Learn | Interesting to read: cart merge on login |
| `16.05-quiz.md` | Check | Reserved |
| `16.06-where-the-project-is-now.md` | Build | Customer logged in; products browsable |
| `16.07-scaffold-cart-module.md` | Build | cart routes, service, schema |
| `16.08-add-to-cart-route.md` | Build | `POST /cart` — productId, storeId, quantity |
| `16.09-get-cart-route.md` | Build | `GET /cart` — populated products |
| `16.10-update-cart-item.md` | Build | `PATCH /cart/:itemId` |
| `16.11-remove-cart-item.md` | Build | `DELETE /cart/:itemId` |
| `16.12-stock-check-on-add.md` | Build | Reject zero stock |
| `16.13-user-scoping-test.md` | Build | User A cannot see User B's cart |
| `16.14-recap-and-whats-next.md` | Wrap | |
| `16.15-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Cart API complete; multi-vendor items; user-scoped.

---

## 17 — Your cart (`17-your-cart/`)

| File | Cluster | What it covers |
|---|---|---|
| `17.01-set-the-scene.md` | Open | Cart needs a face |
| `17.02-grouped-by-store.md` | Open | Visual split before checkout split |
| `17.03-what-youll-build.md` | Open | Cart page + add-to-cart wiring |
| `17.04-where-the-project-is-now.md` | Build | Cart API ready |
| `17.05-cart-page.md` | Build | Items grouped by store; subtotals |
| `17.06-quantity-controls.md` | Build | PATCH on change |
| `17.07-remove-item.md` | Build | DELETE + UI update |
| `17.08-wire-add-to-cart.md` | Build | Product detail → POST /cart |
| `17.09-cart-icon-and-count.md` | Build | Header badge |
| `17.10-verify-cart-persists.md` | Build | Reload page — cart still there |
| `17.11-recap-and-whats-next.md` | Wrap | |
| `17.12-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Cart UI works; items from two stores visible.

---

## 18 — Checkout & orders API (`18-checkout-and-orders/`)

| File | Cluster | What it covers |
|---|---|---|
| `18.01-set-the-scene.md` | Open | One checkout → many vendor orders |
| `18.02-price-snapshot-trap.md` | Open | product_id only → price changes rewrite history |
| `18.03-what-youll-build.md` | Open | Checkout splits cart; snapshots prices |
| `18.04-normalization-payoff.md` | Learn | Why OrderItem embeds priceAtPurchase — *after* they feel the trap |
| `18.05-atomic-stock-decrement.md` | Learn | Race condition; `$inc` with filter |
| `18.06-prime-your-thinking.md` | Learn | Interesting: Stripe idempotency keys |
| `18.07-quiz.md` | Check | Reserved |
| `18.08-where-the-project-is-now.md` | Build | Cart with multi-vendor items |
| `18.09-scaffold-orders-module.md` | Build | orders routes, service |
| `18.10-checkout-route.md` | Build | `POST /checkout` — group by storeId |
| `18.11-snapshot-order-lines.md` | Build | priceAtPurchase, nameAtPurchase, unitAtPurchase |
| `18.12-stock-decrement.md` | Build | Atomic decrement; fail all if any OOS |
| `18.13-clear-cart-on-success.md` | Build | Cart empty after checkout |
| `18.14-vendor-orders-route.md` | Build | `GET /vendor/orders` — scoped + populate customer |
| `18.15-customer-orders-route.md` | Build | `GET /orders/my` |
| `18.16-test-snapshot-immutability.md` | Build | Change product price → old order unchanged |
| `18.17-recap-and-whats-next.md` | Wrap | |
| `18.18-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Checkout splits orders; snapshots verified; stock decrements.

---

## 19 — Track orders (`19-track-orders/`)

| File | Cluster | What it covers |
|---|---|---|
| `19.01-set-the-scene.md` | Open | Bridge from Ch 18 checkout API; Week 3 UI finish; plain English + FAQ |
| `19.02-snapshots-make-ui-easy.md` | Open | No product lookup for prices |
| `19.03-what-youll-build.md` | Open | Customer history + vendor order list UI |
| `19.04-where-the-project-is-now.md` | Build | Checkout API tested |
| `19.05-checkout-page.md` | Build | Review cart → POST /checkout |
| `19.06-order-confirmation-page.md` | Build | Show split orders after checkout |
| `19.07-customer-orders-page.md` | Build | List from `GET /orders/my` |
| `19.08-vendor-orders-page.md` | Build | List from `GET /vendor/orders` |
| `19.09-order-status-badge.md` | Build | pending → packed → dispatched |
| `19.10-vendor-update-status.md` | Build | PATCH fulfilment status |
| `19.11-full-commerce-demo.md` | Build | **Week 3 demo:** browse → cart → checkout → both sides see orders |
| `19.12-recap-and-whats-next.md` | Wrap | |
| `19.13-checklist.md` | Wrap | Definition of Done gate |

**Gate:** End-to-end commerce flow demoable in browser.

---

## 20 — Cache & search (`20-cache-and-search/`)

| File | Cluster | What it covers |
|---|---|---|
| `20.01-set-the-scene.md` | Open | Bridge from Week 3 demo; Redis/cache in plain English; performance layer |
| `20.02-stale-cache-trap.md` | Open | Update price → old cache serves wrong data |
| `20.03-what-youll-build.md` | Open | Redis cache + text index + invalidation |
| `20.04-ttl-vs-event-invalidation.md` | Learn | Both; implement both in sequence |
| `20.05-prime-your-thinking.md` | Learn | Mandatory read: cache invalidation |
| `20.06-quiz.md` | Check | Reserved |
| `20.07-where-the-project-is-now.md` | Build | Full app works; every browse hits MongoDB |
| `20.08-redis-connection.md` | Build | `server/src/shared/redis.js` |
| `20.09-cache-catalogue-middleware.md` | Build | Cache `GET /products` responses |
| `20.10-measure-before-after.md` | Build | Log response times; note in README |
| `20.11-cache-bust-on-product-save.md` | Build | Invalidate on POST/PATCH/DELETE product |
| `20.12-text-index.md` | Build | Index on name + category; `$text` search |
| `20.13-explain-query.md` | Build | `explain()` — index scan not collection scan |
| `20.14-fallback-if-redis-down.md` | Build | Direct MongoDB read; no crash |
| `20.15-recap-and-whats-next.md` | Wrap | |
| `20.16-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Cache works; invalidation verified; search uses index.

---

## 21 — Async jobs & queues (`21-async-jobs-and-queues/`)

| File | Cluster | What it covers |
|---|---|---|
| `21.01-set-the-scene.md` | Open | Bridge from Ch 20; why async after checkout; queues/workers FAQ |
| `21.02-the-sync-trap.md` | Open | Notify-in-handler latency and failure coupling |
| `21.03-what-youll-build.md` | Open | BullMQ + worker process + job contracts |
| `21.04-when-to-queue.md` | Learn | Sync vs async decision table for marketplace |
| `21.05-bullmq-and-redis.md` | Learn | Same Redis as Ch 20 — cache keys vs queue keys |
| `21.06-prime-your-thinking.md` | Learn | Mandatory read: BullMQ docs |
| `21.07-quiz.md` | Check | Reserved |
| `21.08-where-the-project-is-now.md` | Build | Recap: cache + commerce complete |
| `21.09-queue-module.md` | Build | `server/src/shared/queue.js` — BullMQ connection |
| `21.10-worker-process.md` | Build | `server/src/worker.js`; `npm run worker` |
| `21.11-order-created-job.md` | Build | Enqueue from `POST /checkout` — not inline notify |
| `21.12-order-created-processor.md` | Build | Vendor + customer mock notifications (console) |
| `21.13-order-status-job.md` | Build | Enqueue from vendor order status PATCH |
| `21.14-retries-and-idempotency.md` | Build | Attempts, backoff, `jobId` dedup |
| `21.15-verify-async-flow.md` | Build | 201 before worker logs; jobs visible in Redis |
| `21.16-cache-invalidation-job.md` | Build | Optional: async catalogue cache bust |
| `21.17-recap-and-whats-next.md` | Wrap | |
| `21.18-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Checkout enqueues jobs; worker processes notifications; API stays fast; retries documented.

---

## 22 — Deploy (`22-deploy/`)

| File | Cluster | What it covers |
|---|---|---|
| `22.01-set-the-scene.md` | Open | Bridge from localhost; hosting + env vars demystified for first deploy |
| `22.02-secrets-in-git-forever.md` | Open | `git log -- .env` must be empty |
| `22.03-what-youll-build.md` | Open | HTTPS API + client + worker in production |
| `22.04-vps-vs-paas.md` | Learn | Options; pick one path for course |
| `22.05-prime-your-thinking.md` | Learn | Mandatory read: Let's Encrypt / host docs |
| `22.06-quiz.md` | Check | Reserved |
| `22.07-where-the-project-is-now.md` | Build | Full app + worker works locally |
| `22.08-production-env-checklist.md` | Build | All env vars documented in `.env.example` |
| `22.09-build-client-for-production.md` | Build | `npm run build`; set `VITE_API_URL` |
| `22.10-deploy-api.md` | Build | Server on VPS/Render/Railway; PM2 or native |
| `22.11-domain-and-https.md` | Build | DNS + TLS certificate |
| `22.12-deploy-client.md` | Build | Vercel/Netlify/same VPS |
| `22.13-smoke-test-production.md` | Build | Health, login, checkout, worker on live URL |
| `22.14-restart-survival-test.md` | Build | Reboot server — API + worker come back |
| `22.15-recap-and-whats-next.md` | Wrap | |
| `22.16-checklist.md` | Wrap | Definition of Done gate |

**Gate:** Live HTTPS URL; secrets in env only; API and worker survive restart.

---

## 99 — Closing (`99-closing/`)

| File | What it covers |
|---|---|
| `99.01-ship-it.md` | Journey recap in plain English; final ship checklist; live URL deliverable |
| `99.02-document-it.md` | ERD, architecture (API + worker + Redis), DEMO, CHALLENGES — beginner doc guide |
| `99.03-bar-raiser-and-evaluation.md` | Full-journey recap; bar-raiser clause breakdown; viva bank; portfolio framing |

---

## Competency map coverage *(mentor only — never print in mentee content)*

| Competency | Chapters |
|---|---|
| Model vendors, stores, products with ownership | 05 |
| Hash passwords; role-aware JWT; threat model | 06, 07 |
| Vendor cannot touch another vendor's data | 08, 11 |
| Multi-vendor cart; split checkout; price snapshots | 16, 18 |
| Avoid N+1; populate vs aggregate | 14, 18 |
| Images in object storage | 13 |
| Cache catalogue; invalidate on change | 20 |
| Async jobs; checkout off hot path | 21 |
| Deploy; secrets in env | 04, 22 |

---

## Summary stats

| Metric | Count |
|---|---|
| Phases / weeks | 4 (+ intro + closing) |
| Feature chapters | 21 (Ch 02–22) |
| Sub-chapters (total) | ~360 across all chapters |
| Avg sub-chapters per feature chapter | ~12–20 (medium course); guide ideal is 20–30 — denser build steps compensate in early chapters 02–06 |
| Backend + frontend pairs | 06→07, 09→10, 11→12, 14→15, 16→17, 18→19 |
| Concept chapters (shorter build) | 03, 08 (partial) |
| Phase demo milestones | 4 |

---

*This outline follows [`guide.md`](../guide.md): project skeleton first (Ch 02), concepts adjacent to build (Rule 28), backend/frontend paired (§7.6), 20–30 sub-chapters per feature chapter (Rule 24), checklist gate on every chapter (§11).*
