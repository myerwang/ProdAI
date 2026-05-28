---
form: resumable_tus
topic: upload
applies_to: [frontend, backend]
decision: flaky networks, mobile, cross-session resume, large videos
status: stable
last_reviewed: 2026-05-28
---

# Upload 4: Resumable Upload via tus

浏览器上传文件到 tus 兼容的服务器，通过 HTTP PATCH 和偏移量跟踪实现可恢复性。客户端查询当前
上传进度（HEAD 请求），然后从上次中断处恢复（PATCH 缺失字节）。上传 URL 和偏移量由客户端库
持久化到 localStorage 或 IndexedDB，允许跨网络中断、浏览器标签关闭甚至进程重启后恢复。适合
移动网络、弱连接环境、长时间上传和跨会话断点续传场景。

## When to use

- 移动或弱网环境：频繁中断、延迟高、带宽不稳定
- 浏览器可能中途关闭或网络中断：上传 >10 分钟
- 长视频或大媒体文件：>500 MB，需跨会话恢复
- 进程重启后恢复：tab 关闭、浏览器退出、网络故障后仍可续传
- 用户需要暂停和继续功能：显式控制上传状态

## When NOT to use

- 稳定 LAN 和小文件 (≤50 MB)：使用表单 2（直接 PUT）足够且更简单
- 文件 <100 MB 且网络稳定：表单 2（预签名 PUT）或表单 3（并行部分）更轻
- 存储无 tus 网关且不愿部署兼容服务器：表单 3（多部分）或表单 2
- 仅需前向传输，不需要中途暂停：表单 2 或表单 3

## Conclusion

tus 1.0 通过 HTTP PATCH + 偏移量跟踪实现断点续传。(1) 客户端 POST 元数据，服务端返回上传 URL；
(2) 客户端循环 HEAD 查询偏移、PATCH 发送缺失字节、网络中断后 backoff 重试；(3) 偏移量等于文件大小
时移动到 durable key。客户端库通过 fingerprint 避免错误恢复。始终使用 uppy 等 tus 库而非手搓，避免
漏掉并发、offset 漂移、幂等重试。

## Frontend

```
# Types
interface TusUploadSession:
  url: string; fingerprint: string; size: integer; offset: integer

function TusFileUpload(props):
  state file: optional Blob = null
  state uploadState: string = "idle"  # idle | creating | uploading | paused | resuming | done | error
  state progress: float = 0.0
  state errorMsg: optional string = null
  state totalUploaded: integer = 0
  state chunkSize: integer = 1_048_576  # 1 MB

  function onFileSelect(f: Blob):
    file = f
    uploadState = "idle"
    errorMsg = null
    totalUploaded = 0
    # Attempt to resume existing upload or start fresh
    attemptResume(f)

  function attemptResume(f: Blob):
    fingerprint = computeFingerprint(f)
    savedSession = localStorage.get("tus_" + fingerprint)
    if savedSession and savedSession.url:
      uploadState = "resuming"
      verifyAndResume(f, savedSession)
    else:
      uploadState = "creating"
      createUpload(f)

  function computeFingerprint(f: Blob) -> string:
    # name+size+mtime+sha256(first-1MB)
    firstMBSha = await sha256(f.slice(0, 1_048_576))
    return join([f.name, f.size, f.lastModified, firstMBSha], "_")

  function verifyAndResume(f: Blob, savedSession):
    # HEAD with retry; if 404/410 or expired, start fresh; else resume from offset
    async function attemptHead(retries=0):
      try:
        response = await httpRequest({
          method: "HEAD",
          url: savedSession.url,
          headers: {"Tus-Resumable": "1.0.0"}
        })
        
        if response.status == 200 or response.status == 204:
          offset = parseInt(response.headers.get("Upload-Offset") or "0")
          totalSize = parseInt(response.headers.get("Upload-Length") or f.size)
          if offset == totalSize:
            uploadState = "done"
            finalize(f, savedSession.url)
          elif offset >= 0 and offset < totalSize:
            uploadState = "uploading"
            continuePatch(f, savedSession.url, offset)
          return
        
        if response.status == 404 or response.status == 410:
          localStorage.remove("tus_" + fingerprint)
          createUpload(f)
          return
        
        if response.status >= 500 and retries < 3:
          await sleep(min(5000, 500 * pow(2, retries)))
          attemptHead(retries + 1)
          return
        
        throw Error("HEAD failed: " + response.statusText)
      catch e:
        if retries < 3:
          await sleep(min(5000, 500 * pow(2, retries)))
          attemptHead(retries + 1)
        else:
          uploadState = "error"
          errorMsg = "Failed to verify: " + e.message
          localStorage.remove("tus_" + fingerprint)
    
    attemptHead()

  function createUpload(f: Blob):
    try:
      response = await httpRequest({
        method: "POST",
        url: "/files",
        headers: {
          "Tus-Resumable": "1.0.0",
          "Upload-Length": f.size.toString(),
          "Upload-Metadata": encodeMetadata({filename: f.name, filetype: f.type}),
          "Content-Type": "application/json"
        }
      })
      
      if response.status != 201:
        throw Error("Create failed: " + response.statusText)
      
      uploadUrl = response.headers.get("Location")
      if not uploadUrl:
        throw Error("No Location header")
      
      fingerprint = computeFingerprint(f)
      session = {url: uploadUrl, fingerprint: fingerprint, size: f.size, offset: 0}
      localStorage.set("tus_" + fingerprint, session)
      
      uploadState = "uploading"
      continuePatch(f, uploadUrl, 0)
    catch e:
      uploadState = "error"
      errorMsg = e.message

  function continuePatch(f: Blob, uploadUrl: string, currentOffset: integer):
    async function attemptPatch(retries=0):
      try:
        if currentOffset >= f.size:
          uploadState = "done"
          finalize(f, uploadUrl)
          return
        
        endOffset = min(currentOffset + chunkSize, f.size)
        chunk = f.slice(currentOffset, endOffset)
        
        patchResponse = await httpRequest({
          method: "PATCH",
          url: uploadUrl,
          headers: {
            "Tus-Resumable": "1.0.0",
            "Content-Type": "application/offset+octet-stream",
            "Upload-Offset": currentOffset.toString()
          },
          body: chunk,
          onProgress: (loaded) => progress = (currentOffset + loaded) / f.size
        })
        
        if patchResponse.status == 204:
          newOffset = parseInt(patchResponse.headers.get("Upload-Offset") or currentOffset.toString())
          totalUploaded = newOffset
          fingerprint = computeFingerprint(f)
          localStorage.set("tus_" + fingerprint, {
            url: uploadUrl, fingerprint: fingerprint, size: f.size, offset: newOffset
          })
          if newOffset < f.size:
            continuePatch(f, uploadUrl, newOffset)
          else:
            uploadState = "done"
            finalize(f, uploadUrl)
          return
        
        if patchResponse.status >= 500 and retries < 5:
          await sleep(min(30000, 1000 * pow(2, retries)))
          attemptPatch(retries + 1)
          return
        
        throw Error("PATCH: " + patchResponse.statusText)
      catch e:
        if retries < 5 and isNetworkError(e):
          await sleep(min(30000, 1000 * pow(2, retries)))
          attemptPatch(retries + 1)
        else:
          uploadState = "error"
          errorMsg = e.message
    
    attemptPatch()

  function finalize(f: Blob, uploadUrl: string):
    fingerprint = computeFingerprint(f)
    localStorage.remove("tus_" + fingerprint)
    props.onSuccess({url: uploadUrl, size: f.size, name: f.name})

  function pause():
    if uploadState == "uploading":
      uploadState = "paused"

  function resume():
    if uploadState == "paused" and file:
      uploadState = "uploading"
      attemptResume(file)

  function cancel():
    if uploadState in ["uploading", "paused"]:
      fingerprint = computeFingerprint(file)
      localStorage.remove("tus_" + fingerprint)
      uploadState = "idle"
      progress = 0.0
      totalUploaded = 0
      errorMsg = null

  # Render: file-input (idle) → spinner (creating/resuming) → progress-bar + pause/cancel (uploading) → 
  # upload paused + resume/cancel (paused) → error-message + retry/cancel (error) → checkmark (done)
```

## Backend

```
function handleCreateUploadPost(request):
  if request.headers.get("Tus-Resumable") != "1.0.0":
    return 400 "Tus-Resumable must be 1.0.0"
  
  uploadSize = parseInt(request.headers.get("Upload-Length") or -1)
  if uploadSize <= 0 or uploadSize > 5_000_000_000:
    return 413 "Size invalid or exceeds 5 GB"
  
  metadata = decodeMetadata(request.headers.get("Upload-Metadata"))
  contentType = metadata.filetype or "application/octet-stream"
  filename = sanitizeFilename(metadata.filename or "upload")
  
  user = request.user
  stagingId = generateUUID()
  stagingKey = "uploads/staging/" + user.id + "/" + stagingId
  
  try:
    createWritableObject({
      bucket: STAGING_BUCKET,
      key: stagingKey,
      contentType: contentType,
      metadata: {upload_id: stagingId, user_id: user.id, original_name: filename},
      tags: {ttl_days: 7}
    })
    
    storeUploadSession({
      uploadId: stagingId,
      userId: user.id,
      key: stagingKey,
      size: uploadSize,
      expiresAt: now() + 604800
    })
  catch e:
    return 500 "Failed to allocate upload"
  
  uploadUrl = buildUploadUrl(stagingId)
  return 201 Created {
    headers: {
      "Location": uploadUrl,
      "Tus-Resumable": "1.0.0",
      "Tus-Version": "1.0.0",
      "Tus-Max-Size": "5000000000"
    }
  }

function handleHeadUpload(uploadId: string, user):
  session = queryUploadSession(uploadId)
  if not session or session.userId != user.id:
    return 404 Not Found
  
  try:
    currentSize = headObject({bucket: STAGING_BUCKET, key: session.key}).size or 0
  catch e:
    return 500 "Failed to query storage"
  
  if currentSize > session.size:
    return 400 "Offset exceeds upload size"
  
  return 200 OK {
    headers: {
      "Tus-Resumable": "1.0.0",
      "Upload-Length": session.size.toString(),
      "Upload-Offset": currentSize.toString()
    }
  }

function handlePatchUpload(uploadId: string, request, user):
  uploadOffset = parseInt(request.headers.get("Upload-Offset") or "-1")
  if uploadOffset < 0:
    return 400 "Upload-Offset required"
  
  session = queryUploadSession(uploadId)
  if not session or session.userId != user.id:
    return 404 Not Found
  
  currentOffset = (headObject({bucket: STAGING_BUCKET, key: session.key}).size or 0)
  
  if uploadOffset != currentOffset:
    return 409 Conflict {
      headers: {"Tus-Resumable": "1.0.0", "Upload-Offset": currentOffset.toString()}
    }
  
  chunk = request.body
  if uploadOffset + len(chunk) > session.size:
    return 413 "Chunk exceeds remaining size"
  
  try:
    appendToObject({
      bucket: STAGING_BUCKET,
      key: session.key,
      offset: uploadOffset,
      data: chunk
    })
    newOffset = uploadOffset + len(chunk)
    updateUploadSession({uploadId: uploadId, offset: newOffset})
  catch e:
    return 500 "Failed to append chunk"
  
  return 204 No Content {
    headers: {
      "Tus-Resumable": "1.0.0",
      "Upload-Offset": newOffset.toString()
    }
  }

function handleFinalizeUpload(uploadId: string, user):
  session = queryUploadSession(uploadId)
  if not session or session.userId != user.id:
    return 404 Not Found
  
  currentSize = headObject({bucket: STAGING_BUCKET, key: session.key}).size or 0
  if currentSize != session.size:
    return 400 "Upload incomplete: " + currentSize + " / " + session.size
  
  finalKey = "uploads/" + user.id + "/" + generateUUID() + "/" + sanitizeFilename(storageHead.metadata.original_name)
  
  try:
    copyObject({
      fromBucket: STAGING_BUCKET,
      fromKey: session.key,
      toBucket: DURABLE_BUCKET,
      toKey: finalKey,
      contentType: session.contentType
    })
    deleteObject({bucket: STAGING_BUCKET, key: session.key})
    
    recordId = generateId()
    insertUploadRecord({
      id: recordId,
      userId: user.id,
      storageKey: finalKey,
      contentType: session.contentType,
      size: session.size,
      createdAt: now()
    })
    
    deleteUploadSession(uploadId)
    enqueuePostProcess({uploadId: recordId, key: finalKey, type: session.contentType})
  catch e:
    return 500 "Failed to finalize"
  
  return 200 OK {
    id: recordId,
    url: buildPublicUrl(finalKey),
    size: session.size
  }

# Helpers (brief):
# decodeMetadata: parse "key1 <base64-val1>,key2 <base64-val2>" → {key1: val1, key2: val2}
# encodeMetadata: reverse; both standard tus metadata encoding
```

## Contract

**POST /files (Create Upload)**
- Headers: `Tus-Resumable: 1.0.0`, `Upload-Length: <size>`, `Upload-Metadata: filename <b64>, filetype <b64>` (optional)
- Response 201: `Location: /uploads/tus/<id>`, `Tus-Resumable`, `Tus-Version`, `Tus-Max-Size: 5000000000`
- Errors: 411 (no Upload-Length), 413 (size > 5GB)

**HEAD /uploads/tus/<upload-id> (Get Progress)**
- Headers: `Tus-Resumable: 1.0.0`
- Response 200: `Tus-Resumable`, `Upload-Length`, `Upload-Offset`
- Errors: 404 (not found/expired), 403 (not owner)

**PATCH /uploads/tus/<upload-id> (Resume/Append)**
- Headers: `Tus-Resumable: 1.0.0`, `Upload-Offset: <offset>`, `Content-Type: application/offset+octet-stream`
- Body: raw bytes (1 MB chunk typical)
- Response 204: `Tus-Resumable`, `Upload-Offset: <new-offset>`
- Errors: 409 (offset mismatch; server returns current offset), 413 (exceeds size), 404 (expired)

**CORS Exposure** (required):
```
Access-Control-Expose-Headers: Upload-Length, Upload-Offset, Location, Tus-Version, Tus-Resumable, Tus-Max-Size, Tus-Extension
Access-Control-Allow-Methods: GET, POST, PATCH, HEAD, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Upload-Length, Upload-Offset, Upload-Metadata, Tus-Resumable
```

## Pitfalls

- ❌ **手搓 resumable 逻辑**：用 uppy 等库，避免漏掉并发 PATCH、offset 漂移、幂等重试。

- ❌ **未暴露 tus 响应头**：CORS 必须 `Access-Control-Expose-Headers: Upload-Offset, Location, Upload-Length, Tus-Resumable, Tus-Version, Tus-Max-Size` 否则浏览器读不到。

- ❌ **无 size 上限或 quota**：POST 验证 `Upload-Length ≤ 5GB` + per-user quota 防止存储爆炸。

- ❌ **staging 无 TTL**：7 天自动清理未完成上传，否则累积费用。

- ❌ **只在 POST 检查授权**：HEAD/PATCH 都要验证 token 和用户所有权，防止无授权续传。

- ❌ **恢复错误文件**：fingerprint = name+size+mtime+sha256(first-1MB)，恢复前校验 Upload-Length 匹配。

- ❌ **并发 PATCH 导致 offset 漂移**：服务端校验 Upload-Offset == 存储大小，不匹配返回 409；客户端串行化 PATCH。

- ❌ **长时间无探测**：客户端定期 HEAD 验证 URL 有效；服务端每成功 PATCH 都续期 TTL。

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★ (verified 2026-05-28): 生产级 tus 客户端实现，@uppy/tus 插件支持恢复、暂停、取消，提供 fingerprint 和 localStorage 持久化
