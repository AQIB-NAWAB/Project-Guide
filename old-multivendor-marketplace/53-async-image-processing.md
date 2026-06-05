# Chapter 53 — Async image processing

When you built uploads (Chapter 43), you deferred something with a note: raw product photos are large and the wrong size for a catalogue grid, but *generating resized versions* is slow CPU work that shouldn't block the upload. Now you have a reliable, retryable, idempotent job queue (Chapters 50–52) — exactly the tool for it. This chapter generates image thumbnails in the background: the upload stays instant, and a worker turns each raw photo into the sized versions your catalogue needs.

It's the last async *feature* chapter, and it exercises everything: object storage (Chapter 43), the queue, idempotency (Chapter 52), and graceful handling of work that takes real time.

## Where we're headed

By the end, recording an uploaded image enqueues a processing job; a worker fetches the original from object storage, generates thumbnails, stores them back, and updates the image record — all without the upload request waiting, and safe to retry.

## Step 1 — Why image processing is the textbook async job

Resizing an image is the ideal background job, for every reason from Chapter 49:

- **It's slow** — decoding a multi-megabyte photo and generating several sizes takes real CPU time, far more than a request should hold.
- **It's CPU-heavy** — doing it inside the API would block a request worker from serving others; doing it in a dedicated worker keeps the API responsive.
- **The user doesn't need it instantly** — the upload succeeds immediately; thumbnails can appear a moment later.
- **It's retryable** — a transient failure (storage blip) just retries.

So the upload records the image and *enqueues* processing; the worker does the heavy lifting.

## Step 2 — The flow

```
vendor records image (Chapter 43)  →  enqueue 'process-image' { imageId }  →  202, done
                                                    │
worker:  load image by id → fetch ORIGINAL from object storage → generate thumbnails (e.g. small/medium)
         → upload each thumbnail back to storage under derived keys → update the image row (keys, status='ready')
```

The image record starts in a `pending`/`processing` state (Chapter 43) and the worker flips it to `ready` when thumbnails exist. Until then, the catalogue can show the original or a placeholder. The payload is just `imageId` (Chapter 50) — the worker fetches the rest.

## Step 3 — Idempotency for image jobs

Per Chapter 52, assume this job may run twice. Make it idempotent: before generating, check whether thumbnails **already exist** for this image (by their derived keys, or the image's `ready` status) and skip if so. Regenerating identical thumbnails isn't *harmful* (it overwrites with the same output), but checking-and-skipping saves the wasted CPU and makes "ran twice = ran once" explicit. Use the image key as the natural idempotency key.

## Step 4 — Display while processing

A small UX decision: what does the catalogue show between upload and "ready"? Options: show the original (works, just larger), show a placeholder/blur, or hide the image until ready. For this course, showing the original (or a placeholder) while `status != 'ready'`, then swapping to the thumbnail, is fine — the point is the *upload* never waited. Note that the frontend (Chapter 34) reads the image `status` to decide.

## Step 5 — Do it on your project (hands-on)

**1. Enqueue on record.** When `POST /products/:id/images` records an uploaded image (Chapter 43), enqueue `process-image` with the `imageId` and return `202`/`201` immediately.

**2. Build the worker handler** — load the image, fetch the original from object storage, generate one or two thumbnail sizes (a resizing library), upload them back under derived keys, update the image row with the thumbnail keys and `status='ready'`. Make it idempotent (skip if already `ready`).

**3. Add retries** (Chapter 52) so a transient storage failure retries.

**4. Prove the flow:**

```bash
# upload + record an image (Chapter 43 flow) → returns immediately
curl -i -X POST localhost:3000/products/$PID/images -H "authorization: Bearer $V" -d "{\"key\":\"$KEY\"}" -H 'content-type: application/json'   # → 201, status "processing"

# the upload request did NOT wait for thumbnails. Watch the worker generate them:
# worker logs: "processed image <id> → thumbnails ready"

# moments later, the image record is ready with thumbnail keys
psql "$DATABASE_URL" -c "SELECT status, storage_key FROM images WHERE id='<id>';"   # → status 'ready'
# the catalogue now serves the thumbnail
```

The upload returning before thumbnails exist, and the worker producing them a moment later, is async image processing working. Confirm running the job twice doesn't duplicate work (idempotency).

> 💡 **Hint — validate and sanitize images in the worker, not just by content-type.** A file claiming to be `image/jpeg` at upload (Chapter 43) might not be a valid image, or could be a malicious payload. The processing worker — which actually *decodes* the image — is your real validation point: if decoding fails, mark the image failed rather than crashing the job, and consider stripping metadata (EXIF can carry location/PII). Treat the uploaded bytes as untrusted until your library has successfully parsed them.

> **📖 Mandatory read — before Chapter 54.** Read your image library's docs (resizing, format conversion) and a short piece on **responsive image sizes / thumbnails** (search *"image resizing thumbnails web"*). *Required: deriving sensible sizes matters for catalogue performance.*

> **Interesting to read.** Large platforms generate *dozens* of variants per upload (sizes, formats like WebP/AVIF, crops) entirely in background workers — the upload is instant, the variants stream in. Search *"image processing pipeline thumbnails background"* to see how far the pattern you just built scales.

## Definition of Done

Things you can **see or run** — and the gate to Chapter 54:

- [ ] Recording an uploaded image **enqueues** a `process-image` job; the upload request **returns immediately**
- [ ] A worker **fetches the original**, generates **thumbnails**, stores them back, and updates the image to `ready`
- [ ] The job is **idempotent** (skips if already processed) and **retries** on transient failure
- [ ] The catalogue handles the **processing → ready** transition (shows original/placeholder until ready)
- [ ] The worker **validates the image by decoding it**, failing the job gracefully on a bad file
- [ ] You can explain why image processing is an ideal background job

> **✍️ Log it (mandatory).** In `learning-log/53-async-image-processing.md` — **decision** first, then **topics**: **(Decision)** Why generate thumbnails in a worker rather than during the upload request? Why is decoding-in-the-worker the real image-validation point? **(Topics)** (1) Walk the upload → enqueue → process → ready flow. (2) How did you make the image job idempotent, and what's its natural key? (3) What does the catalogue show between upload and ready, and how does it know? (4) Why are slow + CPU-heavy + retryable the signs of a good background job?

*All boxes ticked and the log written? Then continue. Your jobs run in response to events — but some work runs on a *schedule*, with nobody triggering it. Next you make jobs run themselves.*

---

Next: not all background work is triggered by a request — some runs on a clock. Build scheduled jobs. → **[Chapter 54 — Scheduled jobs](54-scheduled-jobs.md)**
