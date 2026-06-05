# Chapter 42 — Images and object storage: the concept

Your marketplace sells **handmade goods** — and people buy those with their eyes. A pottery listing with no photo of the glaze, a scarf with no image of the weave, won't sell. Images aren't a nice-to-have here; they're core to the product. But files are a fundamentally different kind of data from the rows you've been modelling, and storing them the obvious way — in your database — is a mistake that gets expensive fast. This concept chapter decides *how* images live in your system; Chapter 43 builds the upload flow.

It's a concept chapter, so it ends in Key takeaways — but per Rule 23 you'll still produce something real: the image data model and the documented upload flow your next chapter implements.

## What you'll be able to explain

By the end you can explain why image *bytes* don't belong in your database, what object storage is, how the database references a file instead of holding it, and what a **presigned URL** is and why it's the right way to upload and serve files.

## Section 1 — Why not store images in the database

The tempting approach: add an `image` column of type `BLOB`/`bytea` and stuff the file bytes in. It "works," and it's wrong for a photo-heavy marketplace, concretely:

- **It bloats the database.** Product rows balloon from kilobytes to megabytes; backups, replication, and every query over the table get heavier. Your database is tuned for *structured rows and fast lookups*, not for serving multi-megabyte binaries.
- **It's slow and expensive to serve.** Every image view becomes a database read, through your API server, which is the most expensive way to move a file. You can't easily put a CDN in front of it.
- **It doesn't scale.** Databases are the hardest part of your stack to scale; filling them with files you could store elsewhere wastes your scarcest resource.

The professional rule: **the database stores a *reference* to the file (a URL or key); the file itself lives in object storage.**

## Section 2 — What object storage is

**Object storage** (Amazon S3 and the many S3-compatible services — Cloudflare R2, MinIO, Backblaze) is purpose-built for exactly this: storing and serving large numbers of files (objects) cheaply and at massive scale. The vocabulary is small:

- A **bucket** — a container for objects (your marketplace's product images).
- An **object** — one file, identified by a **key** (its path-like name, e.g. `products/<id>/original.jpg`).
- Objects are served directly over HTTP, and a **CDN** can sit in front to serve them fast worldwide.

So a product's image becomes: the *bytes* in a bucket under some key, and in your `products`/`images` table just the **key or URL** — a short string. Your database stays lean; the files live where files belong.

## Section 3 — Don't push files through your API: presigned URLs

Now the key idea that shapes the whole upload flow. The naive way to accept an upload is: the client sends the file to *your API server*, which forwards it to object storage. That makes your server a bottleneck — it ties up memory and bandwidth proxying big files it has no reason to touch.

The professional way is a **presigned URL**: your server (which holds the storage credentials) generates a short-lived, single-purpose URL that authorizes *one specific upload*, and hands it to the client. The client then uploads the file **directly to object storage** using that URL — never through your API. Your server only issues permission and records the resulting key.

```
client → asks your API "I want to upload an image"
your API → returns a presigned PUT URL (valid briefly, for one object)
client → PUTs the file DIRECTLY to object storage (not via your API)
client → tells your API "done — here's the key"
your API → records the key on the product (after checking ownership)
```

Presigned URLs work for **downloads** too: for *private* files, your server issues a short-lived presigned GET URL so only authorized users can fetch them, and the link expires. (Public product images can be served directly/CDN; private files like invoices use expiring presigned GETs.)

## Section 4 — Processing comes later, asynchronously

A raw phone photo is several megabytes and the wrong size for a thumbnail grid. You'll want resized versions — but generating them is slow work that shouldn't block the upload request. That's a job for the background queue (Week 4): the upload returns immediately, and a worker generates thumbnails afterward (Chapter 53). Note the seam now; build it then.

## Section 5 — Design it for your project (hands-on)

**1. Model images.** Decide how images attach to products and add it to your schema design (a migration in Chapter 43): typically an **`images` table** (`id`, `product_id` FK, `storage_key`, `status`, `created_at`) so a product can have *several* images with an order — better than a single `image_url` column. Add it to `SCHEMA-DECISIONS.md` with the reasoning (and note: `product → images` is one-to-many, `ON DELETE CASCADE`, Chapter 11).

**2. Document the upload flow** in `docs/IMAGES.md`: the presigned-upload sequence (Section 3), the validation you'll enforce (allowed content types, max size), public-vs-private serving, and the async-processing seam (Section 4).

**3. Note the local-vs-prod story.** You'll run an S3-compatible store **locally with Docker** (MinIO, alongside Postgres and Redis — Chapter 5) and point at real S3/R2 in production — and because of your config module (Chapter 6), the app won't care which, just like it didn't care which Postgres. Record that.

> **📖 Mandatory read — before Chapter 43.** Read an **intro to object storage / Amazon S3 concepts** (buckets, keys, objects) and **"presigned URLs"** (search those terms). *Required: Chapter 43 builds the presigned-upload flow you just designed.*

> **Interesting to read.** Storing files in the database is such a common early mistake that "don't store images in your DB" is practically a rite of passage — teams have had to do painful migrations to extract gigabytes of `BLOB`s into object storage after the database buckled. Search *"storing images in database vs filesystem vs object storage"* to see the consensus you're starting on the right side of.

## Key takeaways

You can now explain:

- **Image bytes don't belong in the database** — they bloat it, are slow to serve, and waste your hardest-to-scale resource; the DB stores a **reference**, the file lives in **object storage**.
- **Object storage** (S3-compatible) stores files as **objects** in **buckets**, served over HTTP and frontable by a CDN.
- A **presigned URL** lets the client upload **directly** to storage (and download private files with an expiring link) without proxying files through your API.
- Image **processing** (thumbnails) is slow work for a **background worker**, not the upload request.

> **✍️ Log it (mandatory) — this is the gate.** In `learning-log/42-images-concept.md` — **decision** first, then **topics**: **(Decision)** Why store images in object storage with only a reference in the database, rather than the bytes in the DB? Why model images as their own table rather than a single `image_url` column? **(Topics)** (1) What are buckets, objects, and keys? (2) What problem does a presigned URL solve, and how does the upload flow work? (3) How do presigned URLs serve *private* files safely? (4) Why does image processing belong on a background queue?

*Answered all of it, and committed `IMAGES.md`? Then continue. You have the plan — now build the direct-to-storage upload flow.*

---

Next: implement the presigned-upload flow so vendors can add product photos that go straight to object storage. → **[Chapter 43 — Object storage uploads](43-object-storage-uploads.md)**
