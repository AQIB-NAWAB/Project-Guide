# Chapter 43 — Object storage uploads

You've designed the image model and the presigned-upload flow (Chapter 42). Now build it: vendors uploading product photos that go **straight to object storage**, never through your API, with your server only granting permission and recording the result. It's a genuinely different shape of feature from the CRUD you've built — the file never touches your application code — and getting the security right (who may upload what, where) matters because uploads are a classic abuse vector.

This chapter wires up S3-compatible storage locally, builds the two-step presigned flow, and enforces ownership and limits — pulling in isolation (Chapter 29), validation (Chapter 16), and config (Chapter 6).

## Where we're headed

By the end a vendor can upload a product image via a presigned URL directly to object storage, your API records the image key against the (owned) product, content-type and size are constrained, and images serve back for display.

## Step 1 — Run object storage locally

Add an **S3-compatible store to your Docker Compose** (Chapter 5) — **MinIO** is the standard local stand-in for S3, running alongside Postgres and Redis. Create a bucket for product images. In production you'll point at real S3, Cloudflare R2, or similar — and thanks to your config module (Chapter 6), that's just different `S3_*` environment values; your code doesn't change. Add the storage endpoint, bucket, and credentials to your config schema (validated at startup, like everything else).

```
# docker-compose.yml (add) — local S3-compatible storage
minio:
  image: minio/minio
  command: server /data
  ports: ["9000:9000"]
  environment: { MINIO_ROOT_USER: ..., MINIO_ROOT_PASSWORD: ... }   # local dev creds
```

## Step 2 — The two-step presigned upload

The flow from Chapter 42, made concrete as two endpoints:

```
1. POST /products/:id/images/presign   (auth + vendor + OWNS product)
     validate: contentType ∈ allowed image types, size ≤ max
     → server generates a presigned PUT URL for a key like products/<id>/<uuid>.jpg
     → { uploadUrl, key }

2. client PUTs the file directly to uploadUrl  (→ object storage, NOT your API)

3. POST /products/:id/images   (auth + vendor + OWNS product)
     body: { key }
     → server records the image row (product_id, key) after confirming the object exists
     → 201 the image
```

Your API does two small, fast things — issue a scoped permission, then record a key. The multi-megabyte file moves directly between the client and storage. Your server never buffers it.

## Step 3 — Security: the upload is an attack surface

Uploads invite abuse, so enforce limits *at the presign step*, where you still control things:

- **Ownership** (Chapter 29): a vendor may only add images to **their own** product. The presign and record endpoints both check the product belongs to `req.user`'s store — `404` otherwise. (An image attached to someone else's product is an IDOR.)
- **Content type**: constrain the presigned URL to allowed image types (`image/jpeg`, `image/png`, `image/webp`). Don't let users upload executables or HTML (which could be served back and cause XSS).
- **Size limit**: cap the upload size in the presign (object storage can enforce a max via the signed policy). Unbounded uploads are a denial-of-service and a cost bomb.
- **Key naming**: *you* generate the key (`products/<id>/<uuid>`), never trust a client-supplied path — a client-chosen key could overwrite another object or escape the intended prefix.

## Step 4 — Public vs private serving

Decide how images are served (Chapter 42): product images are public, so serve them via a public bucket/CDN URL stored as the key. Private files (later, invoices/payout docs) would use **expiring presigned GET URLs** so only authorized users fetch them. For product images, public direct/CDN serving is right — they're meant to be seen.

## Step 5 — Do it on your project (hands-on)

**1. Add MinIO to Compose** and an `images` bucket; add `S3_*` config (validated, Chapter 6).

**2. Migrate the `images` table** (Chapter 42 design: `product_id` FK with `ON DELETE CASCADE`, `storage_key`, `status`) — new migration (Chapter 12).

**3. Build `POST /products/:id/images/presign`** — guarded by auth + vendor + ownership, validating content type and size, returning a presigned PUT URL and key.

**4. Build `POST /products/:id/images`** — records the key against the owned product.

**5. Upload an image end-to-end and verify ownership:**

```bash
V=$(curl -s -X POST localhost:3000/auth/login -d '...' -H 'content-type: application/json' | jq -r .accessToken)

# 1. get a presigned URL for your own product
PRESIGN=$(curl -s -X POST localhost:3000/products/$MYPID/images/presign -H "authorization: Bearer $V" -d '{"contentType":"image/jpeg","size":204800}' -H 'content-type: application/json')
URL=$(echo $PRESIGN | jq -r .uploadUrl); KEY=$(echo $PRESIGN | jq -r .key)

# 2. upload the file DIRECTLY to storage
curl -X PUT "$URL" --upload-file ./mug.jpg -H 'content-type: image/jpeg'

# 3. record it
curl -i -X POST localhost:3000/products/$MYPID/images -H "authorization: Bearer $V" -d "{\"key\":\"$KEY\"}" -H 'content-type: application/json'   # → 201

# ownership: presign for ANOTHER vendor's product → blocked
curl -i -X POST localhost:3000/products/$OTHERPID/images/presign -H "authorization: Bearer $V" -d '{"contentType":"image/jpeg","size":1000}' -H 'content-type: application/json'   # → 404
```

The file went straight to storage, your API only issued permission and recorded a key, and a vendor can't attach images to another's product. Confirm the catalogue (Chapter 35) now shows image URLs.

> 💡 **Hint — the "orphaned object" problem.** A client can get a presigned URL, upload the file, then *never* call the record endpoint — leaving an object in storage with no database row (an orphan). And a record can exist before the upload finishes. Handle this deliberately: mark images `pending` until confirmed, and run a periodic cleanup (a scheduled job, Chapter 54) that deletes objects with no matching row. Two-step uploads always need a reconciliation story.

> **📖 Mandatory read — before Chapter 44.** Read your storage SDK's **presigned URL** docs and **S3 upload security** guidance (content-type and size restrictions). *Required: upload limits are the difference between a feature and an abuse vector.*

> **Interesting to read.** Misconfigured object storage — public buckets, unrestricted uploads, client-controlled keys — is behind a long list of real data leaks and abuse incidents. Search *"S3 bucket misconfiguration breach"* to see why locking down *who can upload what, where* is as important as the feature itself.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 44:

- [ ] **S3-compatible storage runs locally** (MinIO in Compose); `S3_*` config is validated at startup
- [ ] Uploads use a **presigned URL** — the file goes **directly** to storage, never through your API
- [ ] An `images` table records the **key** against the product (`ON DELETE CASCADE`)
- [ ] Presign/record enforce **ownership** (`404` for another vendor's product), **content type**, and **size limit**; the server generates the key
- [ ] An uploaded image **serves back** for display in the catalogue
- [ ] You can describe the orphaned-object problem and its reconciliation

> **✍️ Log it (mandatory).** In `learning-log/43-object-storage-uploads.md` — **decision** first, then **topics**: **(Decision)** Why upload directly to storage via a presigned URL instead of through your API? Why must the *server* generate the object key rather than trusting the client? **(Topics)** (1) Walk the two-step presigned upload flow. (2) What three limits do you enforce at presign time, and what abuse does each prevent? (3) How does ownership apply to image uploads (tie to Chapter 29)? (4) What is the orphaned-object problem and how do you reconcile it?

*All boxes ticked and the log written? Then continue. Products have photos, prices, and a fast catalogue — now let customers actually *buy*. The commerce module begins.*

---

Next: customers need somewhere to collect what they want before buying — build the multi-vendor cart. → **[Chapter 44 — The cart](44-cart.md)**
