# Upload Taxonomy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new `upload/` topic folder to ProdAI containing 8 production-grade file/image/binary upload patterns, each as a self-contained markdown file with Frontend/Backend/Contract sections and ≥5,000★ OSS references.

**Architecture:** Docs-only delivery. 9 new markdown files in `upload/`, 1 contracting edit to `form/09_file_upload_form.md`, and 3 root README files updated for tri-lingual sync. No code, no build, no tests in the traditional sense — "verification" is grep-based quality-bar checks against AGENTS.md §3.

**Tech Stack:** Markdown + YAML frontmatter only. Pseudocode (no real language). GitHub API (already done for star verification during brainstorming).

**Spec:** [docs/superpowers/specs/2026-05-28-upload-taxonomy-design.md](../specs/2026-05-28-upload-taxonomy-design.md)

---

## Conventions for every NN_*.md file

Each file **MUST** contain, in order:

1. YAML frontmatter:
   ```yaml
   ---
   form: <slug>
   topic: upload
   applies_to: [<frontend and/or backend>]
   decision: <one-sentence "when to pick this">
   status: stable
   last_reviewed: 2026-05-28
   ---
   ```
2. `# Upload N: <Title>` H1
3. `## When to use` — bulleted concrete triggers
4. `## When NOT to use` — bulleted concrete anti-triggers
5. `## Conclusion` — 2–4 sentence punchline, no hedging
6. `## Frontend` — pseudocode block (AGENTS.md §5 rules: no `useState`/`async fn`/SQL)
7. `## Backend` — pseudocode block
8. `## Contract` — table or bullets: endpoint, method, request shape, response shape, status codes, headers, CORS exposure, authz scope
9. `## Pitfalls` — bullet list `❌ <anti-pattern> / **Fix**: ...`; must touch all 6 cross-cutting concerns where relevant (authorization scope, server-side re-validation, orphan objects, idempotency, CORS, backpressure)
10. `## References` — at least one `- [owner/repo](https://github.com/owner/repo) — ~Nk★ (verified 2026-05-28): one-line use` with stars ≥5,000

**Pseudocode discipline (§5):** allowed: `state X: type`, `function`, `effect on (deps)`, `try/catch`, `list of T`, `optional T`, `map of K to V`. Forbidden: `useState`, `useEffect`, `async fn` Rust style, SQL `SELECT ... FROM`.

**Verification helper** (run after each NN file is written):

```bash
FILE=upload/0X_xxx.md
grep -E "^form: " $FILE && \
grep -E "^topic: upload$" $FILE && \
grep -E "^last_reviewed: 2026-05-28$" $FILE && \
grep -E "^## When to use$" $FILE && \
grep -E "^## When NOT to use$" $FILE && \
grep -E "^## Conclusion$" $FILE && \
grep -E "^## Frontend$" $FILE && \
grep -E "^## Backend$" $FILE && \
grep -E "^## Contract$" $FILE && \
grep -E "^## Pitfalls$" $FILE && \
grep -E "^## References$" $FILE && \
grep -E "verified 2026-05-28" $FILE && \
echo "OK: $FILE"
# Forbidden-token sweep (must print NOTHING):
grep -nE "useState|useEffect|async fn |SELECT .* FROM|TODO|TBD|\bfill in\b" $FILE && echo "FAIL forbidden tokens"
```

If any line fails, fix inline before commit.

---

### Task 1: Scaffold `upload/` folder

**Files:**
- Create: `upload/` (directory)

- [ ] **Step 1: Create the folder**

```bash
cd /Users/yohji/Dream/ProdAI
mkdir -p upload
```

- [ ] **Step 2: Verify**

```bash
ls -ld upload && echo "OK"
```

Expected: directory listing prints.

No commit yet — empty folder will be committed alongside Task 2.

---

### Task 2: Write `upload/01_server_proxied_multipart.md`

**Files:**
- Create: `upload/01_server_proxied_multipart.md`

**Frontmatter:**
```yaml
---
form: server_proxied_multipart
topic: upload
applies_to: [frontend, backend]
decision: small files (<=5MB) or strict business validation requiring app-layer transit
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats** (each must appear in the file):

- **When to use:** ≤5 MB; needs server-side business validation/transformation/audit before storage; multi-tenant ACL gating; legacy stacks with no object-storage signing capability.
- **When NOT to use:** >50 MB (proxy becomes RAM/bandwidth bottleneck); pure pass-through to object storage (use 02 instead); high concurrency upload bursts (07 streaming or 02 direct).
- **Conclusion:** Browser POSTs `multipart/form-data` to app server; server parses with streaming multipart parser, validates content (sniff type by magic bytes, not extension), persists to durable storage (or forwards to object store), returns durable handle. Hard cap request size at gateway; use streaming parser, never buffer whole body.
- **Frontend pseudocode beats:** `FormData` build, `XMLHttpRequest` (only it exposes `upload.onprogress` cross-browser) — express in pseudocode, do **not** name `XMLHttpRequest` API verbatim; use `httpRequest with progress` abstraction. Cancel via abort handle. Retry policy: exponential backoff on 5xx / network, give up on 4xx.
- **Backend pseudocode beats:** streaming parser pipeline `request → multipart-parser → per-part stream → content sniffer (first KB) → size enforcer → storage writer → finalize`. Reject on first violation. Return `{id, url, size, sha256}`.
- **Contract beats:** `POST /uploads` `Content-Type: multipart/form-data`; field name fixed (e.g. `file`); 201 `{id, url}`; 413 oversize; 415 type rejected; 422 content sniff mismatch; `Idempotency-Key` header optional but supported.
- **Pitfalls (must include):**
  - ❌ Buffering whole body in memory before validation → OOM. **Fix**: stream parse + early reject.
  - ❌ Trusting client `Content-Type` / extension. **Fix**: magic-byte sniff.
  - ❌ No request size cap at gateway. **Fix**: gateway + app double cap.
  - ❌ No `Idempotency-Key`. **Fix**: dedupe finalize.
  - ❌ Storing under user-controlled filename. **Fix**: server-generated key + content-disposition for download.
  - ❌ No virus scan for untrusted sources. **Fix**: quarantine bucket + async scan before promote.
- **References (≥5k★ verified 2026-05-28):**
  - `expressjs/multer` ~12k★ — Node middleware for `multipart/form-data` parsing
  - `node-formidable/formidable` ~7k★ — streaming multipart parser, memory/disk strategies

- [ ] **Step 1: Write the file** following the conventions above and the beats listed.

- [ ] **Step 2: Run the verification helper**

```bash
FILE=upload/01_server_proxied_multipart.md
# ... (helper from top of plan)
```

Expected: prints `OK: upload/01_server_proxied_multipart.md`, no forbidden tokens.

- [ ] **Step 3: Commit**

```bash
git add upload/01_server_proxied_multipart.md
git commit -m "docs(upload): add 01 server-proxied multipart form"
```

---

### Task 3: Write `upload/02_presigned_direct_put.md`

**Frontmatter:**
```yaml
---
form: presigned_direct_put
topic: upload
applies_to: [frontend, backend]
decision: 5-100MB single-shot direct-to-object-storage; offload bandwidth from app
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** 5–100 MB; app server should not see the bytes; storage is S3-compatible (AWS S3 / GCS / R2 / MinIO / Backblaze B2).
- **When NOT to use:** >100 MB (use 03 multipart); tight per-byte business checks before storage (use 01); storage with no presigned URL primitive.
- **Conclusion:** Server signs a short-TTL PUT URL scoped to a deterministic key + exact `content-type` + `content-length` + optional `sha256`. Client PUTs the bytes directly. Server learns success via a `finalize` callback (client-driven) or storage bucket event (server-driven); never trust client-only signal.
- **Frontend pseudocode beats:** `getPresignedUrl({name, type, size})` → `PUT url with body=file and progress` → `finalize({key})`. Single retry on transient. Cancel via abort handle.
- **Backend pseudocode beats:** `signPut({user, name, type, size})` — generates `key = "uploads/{user}/{uuid}/{safeName}"`, enforces `policy = {max-size, allowed-types, expires=5min}`, returns signed URL + the deterministic key. `finalize({key})` — verifies the key matches caller scope, HEADs the object to confirm size & type, records DB row, optionally enqueues virus-scan / post-process (link to 08).
- **Contract beats:** `POST /uploads/sign` → `{url, key, headers, expiresAt}`; client `PUT url` with exactly the signed headers; `POST /uploads/finalize {key}` → `{id, url}`. CORS: bucket must allow `PUT` from app origin; expose `ETag`.
- **Pitfalls:**
  - ❌ Open-ended signed URL (no size/type lock) → user uploads 50 GB of executables.  **Fix**: bind policy.
  - ❌ Client-chosen key → path traversal / overwrite. **Fix**: server-generated UUID key.
  - ❌ Trusting client `finalize` without `HEAD`. **Fix**: re-verify size/type via storage HEAD.
  - ❌ No bucket lifecycle for unreferenced objects. **Fix**: TTL `uploads/tmp/` prefix.
  - ❌ Missing CORS `Access-Control-Expose-Headers: ETag` → can't read upload result. **Fix**: configure on bucket.
  - ❌ Long-TTL signed URLs leaked in logs. **Fix**: ≤5 min TTL + redact in logs.
- **References:**
  - `aws/aws-sdk-js` ~8k★ — canonical S3 signing reference (Node)
  - `transloadit/uppy` ~31k★ — `@uppy/aws-s3` plugin: production presigned PUT client

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/02_presigned_direct_put.md
git commit -m "docs(upload): add 02 presigned direct PUT form"
```

---

### Task 4: Write `upload/03_multipart_parallel_parts.md`

**Frontmatter:**
```yaml
---
form: multipart_parallel_parts
topic: upload
applies_to: [frontend, backend]
decision: large files (>100MB) needing parallel parts and abort capability
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** >100 MB; want parallel throughput; need abort/cleanup of in-flight uploads; storage exposes a multipart primitive (S3 CreateMultipartUpload / GCS resumable).
- **When NOT to use:** Flaky networks where cross-session resume matters (use 04 tus); files small enough for single PUT (use 02).
- **Conclusion:** Three-phase choreography: `init` (server creates multipart upload, returns `uploadId` + part size), `uploadParts` (client requests per-part signed URLs, uploads parts with bounded concurrency, collects `{partNumber, eTag}`), `complete` (server calls storage `CompleteMultipartUpload` with the parts list). Always offer `abort` to clean up storage-side state and bill.
- **Frontend pseudocode beats:** `splitIntoParts(file, partSize)` → bounded concurrent uploader (worker pool size 3–6) → on each part: get signed URL, `PUT` with progress, store `eTag`. Aggregate progress = sum of per-part bytes. Pause/resume = pause submitting new parts to the pool; in-flight ones finish. Abort calls `/abort`.
- **Backend pseudocode beats:** `initiate({key, type, size})` → storage `CreateMultipartUpload` → `{uploadId, partSize, maxParts}`. `signPart({key, uploadId, partNumber})` → signed `PUT`. `complete({key, uploadId, parts: [{partNumber, eTag}]})` → storage `CompleteMultipartUpload` → returns final object handle. `abort({key, uploadId})` → storage `AbortMultipartUpload`.
- **Contract beats:** `POST /uploads/mp/init`, `POST /uploads/mp/part`, `POST /uploads/mp/complete`, `POST /uploads/mp/abort`. Part size 5 MB–5 GB (S3 limits); max 10,000 parts. `ETag` per part must be persisted by client and echoed back at complete.
- **Pitfalls:**
  - ❌ Forgetting `abort` cleanup → storage holds incomplete multipart state indefinitely + bills. **Fix**: client `abort` on cancel; bucket lifecycle rule for incomplete multipart >24h.
  - ❌ Unbounded part concurrency → connection storm / browser hangs. **Fix**: bounded worker pool (3–6).
  - ❌ Part size <5 MB (S3 minimum, except last) → complete fails. **Fix**: compute part size = `max(5MB, ceil(size/9999))`.
  - ❌ Re-signing entire upload on each retry. **Fix**: idempotent per-part sign by `(uploadId, partNumber)`.
  - ❌ Trusting client part list at complete. **Fix**: server validates total assembled size before responding success.
- **References:**
  - `transloadit/uppy` ~31k★ — `@uppy/aws-s3-multipart`: production parallel-parts client
  - `aws/aws-sdk-js` ~8k★ — S3 multipart server-side API reference

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/03_multipart_parallel_parts.md
git commit -m "docs(upload): add 03 multipart parallel parts form"
```

---

### Task 5: Write `upload/04_resumable_tus.md`

**Frontmatter:**
```yaml
---
form: resumable_tus
topic: upload
applies_to: [frontend, backend]
decision: flaky networks, mobile, cross-session resume, large videos
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** Mobile / poor connectivity / browsers that can close mid-upload; long-running uploads (>30 min); need resume after process restart.
- **When NOT to use:** Stable LAN with small-to-medium files (use 02 / 03); storage has no tus gateway and you don't want to run one (use 03).
- **Conclusion:** tus is an open protocol (HTTP PATCH-based) that tracks upload offset server-side, so a client can `HEAD` to learn how many bytes already arrived and `PATCH` only the missing tail. Resume survives network drops, tab closes, even browser restarts (offset persisted in localStorage/IndexedDB by the client lib). Run a tus server in front of (or terminating to) object storage.
- **Frontend pseudocode beats:** `upload = tusUpload({endpoint, file, metadata, chunkSize, retryDelays})` → on start: `POST /files` (server returns upload URL) → loop: `HEAD url` → `PATCH url at offset` with chunk → on disconnect: backoff + resume from last offset. Persist `{url, file fingerprint}` locally so next session can resume the same upload.
- **Backend pseudocode beats:** `createUpload({size, metadata}) → uploadUrl` (allocates a writable object / staging blob). `head(uploadUrl) → {offset, length}`. `patch(uploadUrl, offset, chunkBytes) → {newOffset}`. On `offset == length`: finalize (move from staging to durable key, emit event for 08). Use a tus-compatible server library that delegates to object storage rather than hand-rolling.
- **Contract beats:** tus 1.0 protocol headers: `Tus-Resumable: 1.0.0`, `Upload-Length`, `Upload-Offset`, `Upload-Metadata`, `Content-Type: application/offset+octet-stream`. CORS expose: `Upload-Offset, Location, Upload-Length, Tus-Version, Tus-Resumable, Tus-Max-Size, Tus-Extension, Upload-Metadata`.
- **Pitfalls:**
  - ❌ Hand-rolling resumable instead of using tus → reinventing edge cases (concurrent PATCH, offset drift). **Fix**: use a tus library.
  - ❌ Forgetting to expose tus headers via CORS. **Fix**: full expose list above.
  - ❌ No upload size cap at server. **Fix**: enforce `Tus-Max-Size` + per-user quota.
  - ❌ Staging objects never GC'd. **Fix**: TTL on staging prefix; abandon-cleanup job.
  - ❌ Authorization checked only at `createUpload`, not on `PATCH`. **Fix**: re-check token on every PATCH.
  - ❌ Resuming a different file under same URL (fingerprint mismatch). **Fix**: client lib fingerprint = `name+size+lastModified+sha256-of-first-MB`.
- **References:**
  - `transloadit/uppy` ~31k★ — `@uppy/tus` plugin, the highest-star tus client integration

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/04_resumable_tus.md
git commit -m "docs(upload): add 04 resumable tus form"
```

---

### Task 6: Write `upload/05_client_preprocessing.md`

**Frontmatter:**
```yaml
---
form: client_preprocessing
topic: upload
applies_to: [frontend]
decision: shrink/normalize before upload to save bandwidth and strip privacy metadata
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** Photo uploads from phones (HEIC → JPEG, downscale to ~2k px long edge); EXIF GPS/serial-strip required for privacy; client-side hash for dedupe before upload; bandwidth-constrained clients.
- **When NOT to use:** Lossy preprocessing forbidden (medical/legal originals); server-side derivation already exists and is cheap (use 08 instead); very-low-power devices where preprocessing CPU > saved bandwidth.
- **Conclusion:** Run a deterministic pipeline in a Web Worker: `decode → resize → re-encode → strip metadata → hash`. Upload the smaller normalized blob via 01/02/03/04. Keep the original only if regulatory; otherwise the normalized form IS the canonical artifact. Always re-validate server-side — preprocessing is UX/bandwidth, not security.
- **Frontend pseudocode beats:** Pipeline in a worker: `decode(blob) → bitmap` (use platform image decoder), `if width > max: resize keeping aspect`, `reencode(bitmap, target=jpeg|webp, quality=0.82) → blob`, `stripMetadata(blob)`, `sha256(blob) → hash`. Cache `hash → key` to skip re-upload of dedupes. HEIC requires explicit decoder polyfill.
- **Backend pseudocode beats:** N/A for this form (applies_to: frontend only) — but state explicitly: "the backend MUST NOT trust the preprocessed shape: re-run type sniff, size cap, and any business-critical derivations server-side."
- **Contract beats:** None — preprocessing is a client-local step. Output blob feeds whichever transfer form (01–04) the app uses.
- **Pitfalls:**
  - ❌ Running heavy decode on main thread → UI jank. **Fix**: Web Worker.
  - ❌ Stripping EXIF orientation without rotating → images sideways. **Fix**: apply orientation transform before strip.
  - ❌ Re-encoding lossless PNG as JPEG silently → quality loss on screenshots/diagrams. **Fix**: detect content type; PNG→PNG, photo→JPEG/WebP.
  - ❌ Treating client hash as authoritative for dedupe in security-sensitive contexts. **Fix**: server re-hashes after receive.
  - ❌ Memory blow-up decoding 40MP photo on low-end mobile. **Fix**: progressive downscale via `createImageBitmap` resize options; cap source dimensions.
- **References:**
  - `pqina/filepond` ~16k★ — file uploader with `filepond-plugin-image-transform` for client-side resize/EXIF
  - `lovell/sharp` ~32k★ — if you keep originals and preprocess server-side as a relay variant

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/05_client_preprocessing.md
git commit -m "docs(upload): add 05 client preprocessing form"
```

---

### Task 7: Write `upload/06_background_offline_queue.md`

**Frontmatter:**
```yaml
---
form: background_offline_queue
topic: upload
applies_to: [frontend]
decision: PWA / offline-first / uploads must survive tab close and flaky mobile networks
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** PWAs with offline expectation; mobile field apps; uploads where "fire-and-forget then close the tab" is desired; combined with 04 tus for resilience-squared.
- **When NOT to use:** Desktop short-lived uploads where in-page progress UX is paramount; environments without Service Worker support.
- **Conclusion:** Persist pending uploads to IndexedDB (file blobs + state machine), register a Service Worker Background Sync, replay queue on connectivity restore. The UI thread is decoupled from the upload — closing the tab does not kill the upload (within SW lifetime). Combine with 04 (tus) to also survive SW termination mid-upload.
- **Frontend pseudocode beats:** Enqueue: `idb.put('uploads', {id, blob, target, state: 'pending', attempts: 0})`. SW: `self.addEventListener('sync', replay)` → `replay()` iterates queue → calls underlying transfer (01/02/03/04) → on success: `idb.delete(id)` + `postMessage` to clients → on transient failure: backoff + re-register sync → on permanent failure (4xx): mark `state='dead'`, notify user. UI subscribes to `BroadcastChannel` to live-update progress.
- **Backend pseudocode beats:** N/A (frontend form). Note: backend MUST be idempotent at finalize (see 02 / 03 / 04) because the SW may replay a successful upload whose response was lost.
- **Contract beats:** Inherits whatever transfer form the SW invokes. Add: `Idempotency-Key` per queued upload, generated at enqueue time, persisted in IndexedDB row.
- **Pitfalls:**
  - ❌ Holding the blob only in memory → tab close loses it. **Fix**: persist blob to IndexedDB at enqueue.
  - ❌ No idempotency key → replay double-creates the resource. **Fix**: stable key generated at enqueue.
  - ❌ Unbounded queue growth on dead uploads. **Fix**: cap attempts (e.g. 10) → `state='dead'`; user-driven retry only.
  - ❌ Notification only via UI thread → user closed the tab. **Fix**: SW `showNotification` on terminal success/failure.
  - ❌ Service Worker storage quota exhaustion. **Fix**: `navigator.storage.estimate()` pre-check; refuse enqueue if <2× file size free.
- **References:**
  - `GoogleChrome/workbox` ~13k★ — Background Sync recipes (`workbox-background-sync`)
  - `transloadit/uppy` ~31k★ — Golden Retriever plugin: cross-tab persistence + restore

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/06_background_offline_queue.md
git commit -m "docs(upload): add 06 background offline queue form"
```

---

### Task 8: Write `upload/07_streaming_server_ingestion.md`

**Frontmatter:**
```yaml
---
form: streaming_server_ingestion
topic: upload
applies_to: [backend]
decision: when proxy is unavoidable, stream don't buffer; bound memory regardless of body size
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** Server proxy is mandatory (compliance, region, gateway), but file size is unbounded and you must not OOM; transcoding/scanning while bytes flow; forwarding to object storage with constant memory.
- **When NOT to use:** Direct-to-storage is allowed (use 02/03/04 — they're strictly cheaper); body is guaranteed small (use 01 with sane parser).
- **Conclusion:** Treat the request body as a `Readable` stream. Pipe through a multipart parser that emits per-part **streams**, then through enforcement stages (size counter, magic-byte sniffer on first KB, virus-scan stream), then into a sink (object storage `Upload` constructed from a stream). Backpressure must propagate: if sink is slow, the request pauses; if source is slow, the sink waits. Never collect into a Buffer.
- **Frontend pseudocode beats:** Same as Form 01 client side; this form is about backend implementation under the same wire protocol.
- **Backend pseudocode beats:**
  ```
  pipeline(
    request,                                  # incoming HTTP body stream
    multipartParser(),                        # emits {field, stream} for each part
    onPart(part):
      counter = countingStream(maxBytes)      # error if exceeded
      sniffer = magicByteSniffer(allowlist)   # error on first KB if disallowed
      scanner = virusScanStream()             # optional; clamav, async
      storage.uploadStream({                  # S3 multipart from stream
        key: deriveKey(),
        body: part.stream.pipe(counter).pipe(sniffer).pipe(scanner)
      })
  )
  on any stage error: abort the upload at storage, return 4xx/5xx
  ```
- **Contract beats:** Same wire as 01 (`POST /uploads` multipart). Add streaming-friendly response: 100-continue support so client can drop early on 4xx pre-flight.
- **Pitfalls:**
  - ❌ Calling `request.body()` / `await req.buffer()` → defeats the whole point. **Fix**: keep stream end-to-end.
  - ❌ Sniffing only after full receive. **Fix**: sniff on first KB and abort the source.
  - ❌ Forgetting to abort storage on early error → orphaned multipart state. **Fix**: storage `Abort` in error handler.
  - ❌ Single huge per-request memory limit at gateway (e.g. nginx `client_max_body_size unlimited`). **Fix**: cap at gateway too.
  - ❌ No timeout on slow-loris uploads. **Fix**: per-connection read timeout + min-rate check.
  - ❌ Virus scan synchronous in-line on hot path. **Fix**: scan inline only for small files; large → quarantine + async.
- **References:**
  - `node-formidable/formidable` ~7k★ — streaming multipart parser
  - `expressjs/multer` ~12k★ — disk/stream storage strategies (`multer-storage-*`)

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/07_streaming_server_ingestion.md
git commit -m "docs(upload): add 07 streaming server ingestion form"
```

---

### Task 9: Write `upload/08_post_upload_pipeline.md`

**Frontmatter:**
```yaml
---
form: post_upload_pipeline
topic: upload
applies_to: [backend]
decision: derive artifacts (thumbnails, transcodes, scans) after upload via event-driven jobs
status: stable
last_reviewed: 2026-05-28
---
```

**Required content beats:**

- **When to use:** Upload is just the entry point; real work is derive (thumbnail, transcode, OCR, virus scan, CDN warm); want decoupling so upload latency is unaffected by processing time.
- **When NOT to use:** Single tiny synchronous transform that fits in the upload request (e.g. one thumbnail under 200 ms — just do it inline).
- **Conclusion:** Object storage emits an event on `PutObject` finalize → fan out to per-concern workers (image derive, video transcode, scan, indexer). Workers write derivatives to a separate prefix (`derived/`) and update a job-status row keyed by the upload id. Client polls or subscribes (WS/SSE) for `status`. All workers are **idempotent** and **retry-safe**; if a derivative already exists with matching source-hash, skip.
- **Frontend pseudocode beats:** N/A — but document the client view: after `finalize`, status is `processing`; client polls `GET /uploads/{id}` or subscribes to a status channel; renders skeletons until `derivatives.*` are ready.
- **Backend pseudocode beats:**
  ```
  on storage event "object created" at "uploads/*":
    publish UploadFinalized{key, size, contentType, sha256}

  worker "image-derive":
    on UploadFinalized where contentType startsWith "image/":
      for variant in [thumb-200, web-1280, avif-1280]:
        if exists(derived/{sha256}/{variant}): continue   # idempotent
        bytes = storage.get(key)
        out   = process(bytes, variant)
        storage.put("derived/{sha256}/{variant}", out)
      jobs.update(uploadId, image_status="ready")

  worker "scan":
    on UploadFinalized:
      verdict = scanner.scan(storage.streamOf(key))
      if verdict == "infected":
        storage.delete(key); jobs.update(uploadId, status="quarantined")
      else:
        jobs.update(uploadId, scan_status="clean")
  ```
- **Contract beats:** `GET /uploads/{id}` → `{status, derivatives: {thumb, web, avif}, scan_status}`. Status channel (SSE): event `upload.status` with same shape.
- **Pitfalls:**
  - ❌ Non-idempotent workers → duplicate derivatives on retry. **Fix**: content-addressed derivative keys (`derived/{sha256}/...`).
  - ❌ Long-running transcode in the upload request. **Fix**: event-driven worker.
  - ❌ No backpressure on job queue → workers OOM. **Fix**: bounded concurrency per worker; visibility timeout > worst-case processing.
  - ❌ Returning upload as "done" before scan completes in security-sensitive contexts. **Fix**: gate by scan status before exposing public URLs.
  - ❌ No retry budget → stuck jobs accumulate. **Fix**: max attempts → dead-letter + alert.
  - ❌ CDN serving the un-scanned original. **Fix**: serve only `derived/*` URLs; originals private.
- **References:**
  - `imgproxy/imgproxy` ~11k★ — on-the-fly image derivation server
  - `h2non/imaginary` ~6k★ — HTTP image processing microservice
  - `lovell/sharp` ~32k★ — embedded image processing for custom workers

- [ ] **Step 1: Write the file**
- [ ] **Step 2: Run verification helper**
- [ ] **Step 3: Commit**

```bash
git add upload/08_post_upload_pipeline.md
git commit -m "docs(upload): add 08 post-upload pipeline form"
```

---

### Task 10: Write `upload/README.md` (index + decision tree + summary)

**File:** `upload/README.md`

**Required structure:**

```markdown
---
topic: upload
applies_to: [frontend, backend, storage]
status: stable
last_reviewed: 2026-05-28
---

# Upload — 生产级文件上传 taxonomy

8 个独立 form，覆盖前端 / 后端 / 客户端预处理 / 服务端派生处理。
覆盖载体：图片、文件、视频、二进制 blob。

## 决策树

(<<verbatim from spec §6>>)

## Summary

| # | Form | When (一句话) | applies_to | Primary OSS ≥5k★ |
|---|------|---------------|-----------|-------------------|
| 1 | [server_proxied_multipart](01_server_proxied_multipart.md) | ≤5MB / 必经应用层 | F+B | expressjs/multer ~12k★ |
| 2 | [presigned_direct_put](02_presigned_direct_put.md) | 5–100MB 直传对象存储 | F+B | aws/aws-sdk-js ~8k★, uppy ~31k★ |
| 3 | [multipart_parallel_parts](03_multipart_parallel_parts.md) | >100MB 并行分片可中止 | F+B | uppy ~31k★ |
| 4 | [resumable_tus](04_resumable_tus.md) | 弱网 / 跨会话恢复 | F+B | uppy ~31k★ |
| 5 | [client_preprocessing](05_client_preprocessing.md) | 上传前 resize / HEIC / EXIF / hash | F | filepond ~16k★, sharp ~32k★ |
| 6 | [background_offline_queue](06_background_offline_queue.md) | PWA / tab 关闭也要传 | F | workbox ~13k★, uppy ~31k★ |
| 7 | [streaming_server_ingestion](07_streaming_server_ingestion.md) | 必须代理但要省内存 | B | formidable ~7k★, multer ~12k★ |
| 8 | [post_upload_pipeline](08_post_upload_pipeline.md) | 上传后派生 / 扫毒 / 转码 | B | imgproxy ~11k★, sharp ~32k★ |

## 跨切关注点（每个 form 文件的 Pitfalls 都覆盖）

- Authorization scope — presigned URL 必须 scope 到 key prefix + content-type + size + 短 TTL
- Server-side re-validation — 客户端校验仅 UX；服务端按内容 sniff 类型 / size / 病毒扫描 / image-bomb
- Orphan objects — 未引用对象 TTL/lifecycle 清理；cancel 时显式 DELETE
- Idempotency — finalize 调用必须带幂等键
- CORS — 直传场景 `Access-Control-Expose-Headers: ETag` + 完整 tus header
- Backpressure / limits — 并发上限、单用户带宽、总对象数配额

## 与 `form/` 的关系

[`form/09_file_upload_form.md`](../form/09_file_upload_form.md) 是"作为表单字段"
视角的 1 段总览，详细机制全部在本目录。
```

- [ ] **Step 1: Write the file** with the structure above; copy decision tree verbatim from spec §6.

- [ ] **Step 2: Verify metadata + every NN link resolves**

```bash
FILE=upload/README.md
grep -E "^topic: upload$" $FILE
grep -E "^last_reviewed: 2026-05-28$" $FILE
for n in 01 02 03 04 05 06 07 08; do
  grep -q "${n}_" $FILE && test -f "upload/${n}_"*.md && echo "OK $n" || echo "FAIL $n"
done
```

Expected: 8 lines `OK 01` … `OK 08`.

- [ ] **Step 3: Commit**

```bash
git add upload/README.md
git commit -m "docs(upload): add taxonomy README with decision tree and summary"
```

---

### Task 11: Shrink `form/09_file_upload_form.md` to a pointer

**File:** `form/09_file_upload_form.md`

**New content** (replace entire file with this):

```markdown
---
form: file_upload_form
topic: form
applies_to: [frontend, backend]
field_set: file(s) / binary
decision: collect file(s)/binary as a form field; for transfer mechanics see upload/
status: stable
last_reviewed: 2026-05-28
---

# Form 9: File Upload Form

A form field whose value is one or more files. From the **form's** perspective the
contract is simple: upload bytes first, hold a reference (id/handle), submit
references — never bytes — with the rest of the form payload. Block submit while
any upload is in flight.

For the actual transfer mechanism (presigned direct PUT vs multipart parallel
parts vs resumable tus vs server-proxied multipart, plus client preprocessing,
offline queue, streaming ingestion, post-upload derivation), see the dedicated
taxonomy: **[../upload/README.md](../upload/README.md)**.

## Form-side contract

- Upload starts on file select; field value = list of done handles
- Validate type/size client-side for UX; server re-validates (see `upload/`)
- Block form submit while any upload status is `uploading`
- On submit, send handles only — never the bytes
- On cancel/remove, also delete the orphaned object (or rely on lifecycle TTL)

## References

For OSS implementations of the upload mechanisms themselves, see
[../upload/README.md](../upload/README.md). Form-side wrappers:

- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★ (verified 2026-05-28): file uploader with dashboard UI, full transfer-mechanism coverage
- [pqina/filepond](https://github.com/pqina/filepond) — ~16k★ (verified 2026-05-28): drop-in upload field with chunking and preview
```

- [ ] **Step 1: Replace the file** with the content above.

- [ ] **Step 2: Verify the redirect works**

```bash
grep -q "../upload/README.md" form/09_file_upload_form.md && \
test -f upload/README.md && echo "OK link target exists"
```

Expected: `OK link target exists`.

- [ ] **Step 3: Commit**

```bash
git add form/09_file_upload_form.md
git commit -m "docs(form): shrink 09 to pointer; transfer mechanics moved to upload/"
```

---

### Task 12: Tri-lingual README sync (`README.md`, `README.ja.md`, `README.en.md`)

**Files:**
- Modify: `README.md` (中文 main)
- Modify: `README.ja.md`
- Modify: `README.en.md`

- [ ] **Step 1: Read all three to find the directory list section**

```bash
for f in README.md README.ja.md README.en.md; do
  echo "=== $f ==="
  grep -nE "^(- |#|##) " $f | head -50
done
```

Identify where each lists the topic folders (e.g. `auth/`, `form/`, `table/`).

- [ ] **Step 2: Insert `upload/` entry into each README**

Inserted alphabetically after `table/` (so order becomes auth → form → table → upload).

**README.md** (中文) — add line in the directory list:
```
- [upload/](upload/) — 文件上传 8 种生产级形式（直传 / 分片 / tus / 客户端预处理 / 后台队列 / 流式 / 派生处理）
```

**README.ja.md** — same line in 日本語:
```
- [upload/](upload/) — ファイルアップロード 8 形式（直送り / 分割並列 / tus 再開 / クライアント前処理 / バックグラウンド / ストリーミング / 後処理）
```

**README.en.md** — same line in English:
```
- [upload/](upload/) — 8 production-grade upload patterns (direct PUT / parallel multipart / tus resumable / client preprocessing / background queue / streaming / post-processing)
```

If each README has a per-topic short description elsewhere, mirror that pattern as well; if there is none for `auth/`/`table/`, do not invent one.

- [ ] **Step 3: Verify all three updated**

```bash
for f in README.md README.ja.md README.en.md; do
  grep -q "upload/" $f && echo "OK $f" || echo "FAIL $f"
done
```

Expected: 3 lines `OK`.

- [ ] **Step 4: Commit**

```bash
git add README.md README.ja.md README.en.md
git commit -m "docs(readme): tri-lingual sync — add upload/ taxonomy entry"
```

---

### Task 13: Final taxonomy-wide verification

- [ ] **Step 1: Run full sweep**

```bash
cd /Users/yohji/Dream/ProdAI
fail=0
for f in upload/*.md; do
  grep -qE "^topic: upload$" "$f" || { echo "FAIL topic: $f"; fail=1; }
  grep -qE "^last_reviewed: 2026-05-28$" "$f" || { echo "FAIL date: $f"; fail=1; }
  grep -qE "^## References" "$f" || { echo "FAIL refs section: $f"; fail=1; }
  grep -qE "verified 2026-05-28" "$f" || { echo "FAIL verified token: $f"; fail=1; }
  # No forbidden concrete-language tokens:
  if grep -nE "useState|useEffect|async fn |^SELECT .* FROM|TODO|TBD" "$f"; then
    echo "FAIL forbidden tokens in $f"; fail=1
  fi
done
test $fail -eq 0 && echo "ALL OK"
```

Expected: `ALL OK`.

- [ ] **Step 2: Confirm star floors with a quick re-eyeball**

```bash
grep -E "~[0-9]+k★" upload/*.md | awk -F'~' '{print $2}' | sort -u
```

Manually scan: every printed number's leading digit + `k` ≥ 5 (i.e. `6k★`, `7k★`, ..., `31k★`, `32k★`). Any `~Nk★` with N < 5 is a violation — locate and replace.

- [ ] **Step 3: Tell the user the push status (per AGENTS.md §7)**

After all green:

> 全部 9 个 upload/ 文件 + form/09 收缩 + 三语 README 同步完成，本地 commits 已就绪。即将 push to main，请确认。

Wait for user OK before `git push`. Do **not** push silently.

---

## Spec coverage check (self-review)

Spec §3 (folder layout)         → Tasks 1, 2–9, 10 ✅
Spec §4 (8 forms + ≥5k★ refs)   → Tasks 2–9, each lists ≥1 verified ref ✅
Spec §5 (file inner structure)  → Conventions header + each task's "Required content beats" ✅
Spec §6 (decision tree)         → Task 10 ✅
Spec §7 (README skeleton)       → Task 10 ✅
Spec §8 (cross-cutting concerns)→ Each Task 2–9 Pitfalls includes the 6 concerns where relevant ✅
Spec §9 (tri-lingual README)    → Task 12 ✅
Spec §10 (deliverables checklist) → Tasks 1–12 ✅
Spec §11 (non-goals)            → Honored: no real code, no cloud-vendor-specific deep dives ✅
