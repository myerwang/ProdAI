---
form: file_upload_form
topic: form
applies_to: [frontend, backend]
field_set: file(s) / binary
decision: collect file(s)/binary with progress, validation, and resilient transfer
status: stable
last_reviewed: 2026-05-28
---

# Form 9: File Upload Form

Collects one or more files / binary blobs as part of a form. Needs progress feedback,
client-side validation (type/size), resilient transfer (chunked/resumable for large files),
and a clear relationship between the upload and the rest of the form payload.

## When to use

- A field's value is a file: avatar, document, media, attachment, import
- Large files needing progress, pause/resume, retry
- Multiple files, drag-and-drop, or direct-to-storage uploads

## When NOT to use

- Tiny, optional, always-fast uploads where a plain file input suffices (still validate)
- Pure text/structured input → Form 1

## Conclusion

**Decouple upload from form submit**: upload files first (ideally direct-to-storage via a
presigned URL), keep the returned file id/handle in the form state, submit the form with
*references* not bytes. Validate type/size on the client AND the server. Chunked/resumable
for large files.

## Pseudocode

```
function FileUploadField(props):                # one field inside a Form 1/2
  state items: list of UploadItem = []          # { id, name, size, progress, status, handle }

  function onSelect(files):                       # from input or drag-drop
    for f in files:
      err = validate(f)                           # type allowlist + max size, CLIENT-side
      if err: addItem(f, status="rejected", error=err); continue
      item = addItem(f, status="uploading", progress=0)
      upload(item, f)

  async function upload(item, f):
    try:
      # preferred: direct-to-storage, server only signs the request
      target = await props.getPresignedUrl({ name: f.name, type: f.type, size: f.size })
      if f.size > CHUNK_THRESHOLD:
        handle = await chunkedUpload(target, f, onProgress = p => setProgress(item, p))
      else:
        handle = await putWithProgress(target, f, onProgress = p => setProgress(item, p))
      setItem(item, status="done", handle=handle)
      props.onChange(doneHandles(items))          # form value = list of file handles/ids
    catch e:
      setItem(item, status="error")               # offer retry; keep other uploads intact

  function removeItem(item):
    cancelIfInFlight(item)
    deleteItems(item)
    props.onChange(doneHandles(items))

  render:
    dropzone + file input → onSelect
    for item in items:
      row: name, size, progress bar, status
        if uploading: cancel button
        if error: retry button
        if done: remove button
    # the parent form blocks submit while any item.status == "uploading"

# parent form submit sends references, not bytes:
#   submit({ ...fields, attachments: form.attachments })   # ids/handles only
```

## Chunked / resumable transfer

```
large file:
  split into N chunks
  upload chunks (optionally parallel, bounded concurrency)
  track which chunks acked → on failure/refresh, resume missing chunks only
  server assembles; finalize call returns the durable handle
prefer a battle-tested protocol/lib (e.g. tus resumable) over hand-rolling
```

## Validation & security (client is not enough)

- Client checks type/size for UX; the **server must re-check** type (by content, not just
  extension/MIME header), size, and scan/quarantine untrusted files.
- Use a type allowlist, not a blocklist. Cap size before upload starts.
- Direct-to-storage presigned URLs must be scoped (key prefix, content-type, size, short TTL).

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★: resumable (tus), chunked, direct-to-storage, progress/retry
- [pqina/filepond](https://github.com/pqina/filepond) — ~16k★: drag-drop uploads with validation, chunking, image preview
- [react-dropzone/react-dropzone](https://github.com/react-dropzone/react-dropzone) — ~11k★: drag-and-drop file selection + client-side accept/size filtering

## Pitfalls / Anti-patterns

- ❌ Bundling file bytes into the form's JSON submit → huge payloads, timeouts, memory blow-up
  - **Fix**: upload first, submit references (ids/handles) only
- ❌ Validating type by extension or client MIME only → spoofable, unsafe
  - **Fix**: server validates by content sniffing; allowlist; scan untrusted files
- ❌ Routing large uploads through your app server → bandwidth + memory bottleneck
  - **Fix**: direct-to-storage with scoped presigned URLs; server only signs
- ❌ No chunking/resume for large files → a blip restarts a 2GB upload from zero
  - **Fix**: chunked + resumable (e.g. tus); resume only missing chunks
- ❌ Letting the form submit while uploads are still in flight → references missing/incomplete
  - **Fix**: block submit until all uploads are `done`; reflect status in the button
- ❌ No cancel/remove → user can't undo a wrong or huge file
  - **Fix**: cancel in-flight, remove completed (and delete the orphaned object server-side)
- ❌ Orphaned uploaded objects when the form is abandoned
  - **Fix**: lifecycle/TTL on un-referenced objects; clean up on cancel/expiry
