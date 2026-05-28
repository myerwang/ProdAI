---
form: server_proxied_multipart
topic: upload
applies_to: [frontend, backend]
decision: small files (<=5MB) or strict business validation requiring app-layer transit
status: stable
last_reviewed: 2026-05-28
---

# Upload 1: Server-Proxied Multipart

Browser sends `multipart/form-data` to app server; server parses the stream, validates content
(sniff type by magic bytes, not extension), enforces size caps, and persists to durable storage
(database, object store, or file system). Returns a durable handle. Suitable for files requiring
server-side business logic, multi-tenant gating, or transformation before storage.

## When to use

- Small files (≤5 MB) that need server-side business validation, transformation, or audit
- Multi-tenant or role-gated upload requiring app-layer ACL enforcement
- Legacy stacks with no object-storage signing capability (presigned URLs)
- Compliance scenarios needing app-layer logging/tampering-detection

## When NOT to use

- Large files (>50 MB): app server becomes a RAM/bandwidth bottleneck
- Pure pass-through to cloud storage: use Form 2 (direct upload) instead
- High-concurrency upload bursts (>50 concurrent): streaming or resumable more efficient
- Files requiring resume/chunking: use Form 7 (streaming resumable) instead

## Conclusion

Multipart upload via app server works well for small, validated files. Browser builds a
`FormData` payload, POSTs to app server, server streams-parses the parts (never buffering the
whole body), validates content by magic-byte sniff, enforces size and type policies, writes to
durable storage, and returns a handle (`{id, url, size, sha256}`). Hard cap request size at
the gateway; use streaming multipart parsers (not memory buffering); deduplicate finalize calls
via `Idempotency-Key` header.

## Frontend

```
interface UploadState:
  idle | uploading | error | done

function FileUploadWidget(props):
  state file: optional Blob = null
  state uploadState: UploadState = "idle"
  state progress: float = 0.0            # 0.0 to 1.0
  state errorMsg: optional string = null

  function onFileSelect(f: Blob):
    file = f
    uploadState = "idle"
    errorMsg = null
    upload(f)

  function upload(f: Blob):
    uploadState = "uploading"
    progress = 0.0
    errorMsg = null

    formData = new FormData()
    formData.append("file", f)
    formData.append("name", f.name)

    retries = 0
    maxRetries = 3
    backoffMs = 1000

    async function attemptUpload():
      try:
        response = await httpRequest({
          method: "POST",
          url: "/uploads",
          body: formData,
          headers: {
            "Idempotency-Key": generateUUID()
          },
          onProgress: (loaded, total) => {
            progress = loaded / total
          }
        })
        if response.status == 201:
          uploadState = "done"
          props.onSuccess(response.json())
          return
        if response.status >= 500:
          raise NetworkError(response.statusText)
        if response.status == 413:
          uploadState = "error"
          errorMsg = "File exceeds max size"
          return
        if response.status == 415:
          uploadState = "error"
          errorMsg = "File type not allowed"
          return
        if response.status == 422:
          uploadState = "error"
          errorMsg = "File content does not match declared type"
          return
        uploadState = "error"
        errorMsg = response.statusText
      catch e:
        if retries < maxRetries and (e instanceof NetworkError):
          retries++
          await sleep(backoffMs)
          backoffMs = min(30000, backoffMs * 2)
          attemptUpload()
        else:
          uploadState = "error"
          errorMsg = e.message

    attemptUpload()

  function cancel():
    if uploadState == "uploading":
      httpRequest.abort()
      uploadState = "idle"
      progress = 0.0

  render:
    if uploadState == "idle":
      file-input (accept any, validation on select)
    if uploadState == "uploading":
      progress-bar (progress)
      cancel-button
    if uploadState == "error":
      error-message (errorMsg)
      retry-button
    if uploadState == "done":
      success-checkmark
```

## Backend

```
interface UploadRequest:
  file: stream of bytes
  name: string

function handleUploadPost(request):
  # Step 1: cap request size at gateway + app
  if request.contentLength > 5_000_000:
    return 413 Payload Too Large

  # Step 2: stream-parse multipart body
  parser = createMultipartStreamParser(request.body)
  filePart = null
  extraFields = {}

  for part in parser.parts():
    if part.fieldName == "file":
      filePart = part
    else:
      extraFields[part.fieldName] = part.asString()

  if not filePart:
    return 400 Missing file field

  # Step 3: open storage writer (lazy, don't buffer)
  hasher = Sha256()
  sizeBytes = 0
  storage_fd = null

  try:
    storage_fd = openTempFile()
    
    # Step 4: sniff magic bytes (first KB), validate type
    sniffBuf = []
    for chunk in filePart.stream:
      sniffBuf.append(chunk)
      if totalBytes(sniffBuf) >= 1024:
        break

    declaredType = filePart.contentType or "application/octet-stream"
    sniffedType = sniffMagicBytes(concatenate(sniffBuf))
    
    if not isAllowedType(sniffedType):
      return 415 Unsupported Media Type

    if declaredType and sniffedType and not typesMatch(declaredType, sniffedType):
      return 422 Content type mismatch

    # Step 5: stream write rest of file, enforce size cap
    for chunk in filePart.stream:
      if sizeBytes + len(chunk) > 5_000_000:
        return 413 Payload Too Large
      write(storage_fd, chunk)
      hasher.update(chunk)
      sizeBytes += len(chunk)

    close(storage_fd)

    # Step 6: finalize & durably persist
    sha256 = hasher.finalize()
    finalId = generateId()
    finalUrl = storeFile(storage_fd.path, finalId, sniffedType)

    return 201 {
      id: finalId,
      url: finalUrl,
      size: sizeBytes,
      sha256: sha256
    }

  catch e:
    if storage_fd:
      close(storage_fd)
      deleteFile(storage_fd.path)
    raise e

function isAllowedType(contentType: string) -> bool:
  # whitelist, e.g. image/*, application/pdf, application/json
  allowedPrefixes = ["image/", "application/pdf", "application/json"]
  for prefix in allowedPrefixes:
    if startsWith(contentType, prefix):
      return true
  return false

function typesMatch(declared: string, sniffed: string) -> bool:
  # loose check: if declared is image/*, accept any image/*
  # if declared is application/pdf, accept application/pdf only
  if endsWith(declared, "/*"):
    return startsWith(sniffed, removeSuffix(declared, "/*"))
  return declared == sniffed or sniffed contains declared
```

## Contract

- **Endpoint**: `POST /uploads`
- **Content-Type**: `multipart/form-data`
- **File field name**: `file` (fixed; optional: `name` field for human-readable filename)
- **Response 201 Created**:
  ```json
  {
    "id": "<durable-handle>",
    "url": "<absolute-or-relative-path>",
    "size": 12345,
    "sha256": "<hex-digest>"
  }
  ```
- **Response 400 Bad Request**: missing file field or parse error
- **Response 413 Payload Too Large**: content exceeds server limit (5 MB example)
- **Response 415 Unsupported Media Type**: file type not in allowlist after magic-byte sniff
- **Response 422 Unprocessable Entity**: declared Content-Type does not match sniffed type
- **Optional header `Idempotency-Key`**: UUID or unique string; server deduplicates finalize by this value (return cached `{id, url}` if same key seen within TTL)

## Pitfalls

- ❌ **Buffering the whole request body in memory before validation**
  - **Fix**: Use a streaming multipart parser. Parse parts on arrival, validate & write as you stream.

- ❌ **Trusting client-supplied `Content-Type` header or file extension**
  - **Fix**: Always sniff magic bytes (first 512 bytes) and validate against allowlist. Reject if declared type does not match sniffed type.

- ❌ **No request size cap at the gateway**
  - **Fix**: Set `Content-Length` limit at reverse proxy / API gateway (e.g. nginx `client_max_body_size`); app also enforces cap as second layer.

- ❌ **No `Idempotency-Key` / deduplication for finalize**
  - **Fix**: Store finalize result keyed by Idempotency-Key; return cached result if replayed within TTL (e.g. 1 hour). Prevents orphaned objects on retry.

- ❌ **Storing file under user-controlled filename**
  - **Fix**: Server generates the storage key; store user-supplied name separately in metadata. Return `Content-Disposition: attachment; filename="<original-name>"` on download if needed.

- ❌ **No anti-virus / malware scan for untrusted uploads**
  - **Fix**: Quarantine uploaded file in a separate bucket; run async scan (e.g. ClamAV via SQS worker); promote to production bucket only after scan clears.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [expressjs/multer](https://github.com/expressjs/multer) — ~12k★ (verified 2026-05-28): Node.js middleware for multipart/form-data parsing with streaming and field/file size limits
- [node-formidable/formidable](https://github.com/node-formidable/formidable) — ~7k★ (verified 2026-05-28): fast streaming multipart parser with memory and disk strategies, no external dependencies
