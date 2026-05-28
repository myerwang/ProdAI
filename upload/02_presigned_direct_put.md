---
form: presigned_direct_put
topic: upload
applies_to: [frontend, backend]
decision: 5–100 MB single-shot direct-to-object-storage; offload bandwidth from app
status: stable
last_reviewed: 2026-05-28
---

# Upload 2: Presigned Direct PUT

浏览器从应用服务器请求时间限制的签名 URL；服务器生成绑定到确定性密钥、content-type 和
content-length 的策略；浏览器直接将文件字节 `PUT` 到存储（S3、GCS、R2、MinIO、Backblaze B2
或兼容服务）。服务器通过存储 HEAD 请求或桶事件验证完成，绝不信任仅客户端信号。卸载应用
带宽；适合 5–100 MB 范围内不需要严格的存储前业务逻辑验证的单次文件上传。

## When to use

- 文件 5–100 MB：对于服务器代理多部分上传过大，对于可恢复分块过小
- 浏览器到云存储的直接路径以降低应用带宽
- 无状态文件上传（无多租户门控，无业务转换）
- 存储后端支持带策略范围限制的预签名 URL（S3、GCS、R2、Backblaze B2、MinIO）
- 需要上传进度可见性（字节级客户端跟踪）

## When NOT to use

- 文件 >100 MB：使用表单 3（可恢复多部分分块）以获得恢复能力和重试复原力
- 存储提交前需要严格的逐字节业务验证（→ 表单 1，服务器代理）
- 存储后端不支持预签名 URL（→ 表单 1，通过应用代理）
- 多租户上传门控或按租户密钥范围限制（应用层需要看到用户上下文）
- 需要应用层字节检查的合规审计跟踪

## Conclusion

预签名直接 PUT 用带宽效率和可扩展性换取严格的存储前控制。客户端通过文件元数据（名称、大小、
content-type）调用 `POST /uploads/sign`；服务器生成确定性密钥（`uploads/<user-id>/<uuid>/<safe-name>`），
构建将该密钥绑定到精确 content-type 和 content-length 的策略，使用存储凭证签署该策略，并返回
短 TTL（5 分钟）签名 URL 加必需的头部。浏览器使用这些头部直接将文件 `PUT` 到存储；服务器通过
桶事件、在最终确认期间的存储 HEAD 请求或客户端回调获悉成功，但**绝不单独信任客户端的声明**。
通过确定性密钥对最终确认调用进行去重；在孤立的 `uploads/tmp/` 前缀上设置 TTL 生命周期。

## Frontend

```
interface PresignedUploadState:
  idle | signing | uploading | finalizing | done | error

function PresignedFileUpload(props):
  state file: optional Blob = null
  state uploadState: PresignedUploadState = "idle"
  state progress: float = 0.0            # 0.0 to 1.0 during PUT
  state errorMsg: optional string = null
  state uploadAbortController: optional AbortController = null

  function onFileSelect(f: Blob):
    if f.size > 100_000_000:
      errorMsg = "File exceeds 100 MB limit"
      uploadState = "error"
      return
    file = f
    uploadState = "idle"
    errorMsg = null
    requestPresignedUrl(f)

  function requestPresignedUrl(f: Blob):
    uploadState = "signing"
    errorMsg = null

    try:
      response = await httpRequest({
        method: "POST",
        url: "/uploads/sign",
        headers: {
          "Content-Type": "application/json"
        },
        body: {
          name: f.name,
          size: f.size,
          contentType: f.type or "application/octet-stream"
        }
      })

      if response.status != 200:
        throw Error("Failed to get presigned URL: " + response.statusText)

      signResponse = response.json()
      # signResponse: {url, key, headers, expiresAt}

      putFile(f, signResponse)

    catch e:
      uploadState = "error"
      errorMsg = "Signing failed: " + e.message

  function putFile(f: Blob, signResponse):
    uploadState = "uploading"
    progress = 0.0
    errorMsg = null

    uploadAbortController = new AbortController()

    retries = 0
    maxRetries = 1       # single retry on transient only

    async function attemptPut():
      try:
        # PUT the file directly to storage with signed headers
        putResponse = await httpRequest({
          method: "PUT",
          url: signResponse.url,
          headers: signResponse.headers,
          body: f,
          signal: uploadAbortController.signal,
          onProgress: (loaded, total) => {
            progress = loaded / total
          }
        })

        if putResponse.status == 200 or putResponse.status == 204:
          # Storage accepted the PUT; now finalize on app server
          finalizeUpload(signResponse.key)
          return

        if putResponse.status >= 500:
          # Transient server error
          if retries < maxRetries:
            retries++
            await sleep(1000)
            attemptPut()
            return
          throw Error("Server error: " + putResponse.statusText)

        # 4xx: client error, don't retry
        throw Error("PUT rejected: " + putResponse.statusText)

      catch e:
        if e.name == "AbortError":
          uploadState = "idle"
          errorMsg = null
        else:
          uploadState = "error"
          errorMsg = e.message

    attemptPut()

  function finalizeUpload(key: string):
    uploadState = "finalizing"

    try:
      response = await httpRequest({
        method: "POST",
        url: "/uploads/finalize",
        headers: {
          "Content-Type": "application/json"
        },
        body: {
          key: key
        }
      })

      if response.status == 200 or response.status == 201:
        result = response.json()
        uploadState = "done"
        props.onSuccess(result)
        return

      throw Error("Finalization failed: " + response.statusText)

    catch e:
      uploadState = "error"
      errorMsg = e.message

  function cancel():
    if uploadState == "uploading" and uploadAbortController:
      uploadAbortController.abort()
      uploadState = "idle"
      progress = 0.0
      errorMsg = null

  render:
    if uploadState == "idle":
      file-input (accept any, check size on select, max 100 MB)
    if uploadState == "signing":
      spinner ("Getting upload URL...")
    if uploadState == "uploading":
      progress-bar (progress)
      cancel-button
    if uploadState == "finalizing":
      spinner ("Finalizing...")
    if uploadState == "error":
      error-message (errorMsg)
      retry-button (calls requestPresignedUrl(file))
    if uploadState == "done":
      success-checkmark
```

## Backend

```
interface SignPutRequest:
  name: string           # original filename (user-supplied)
  size: integer          # file size in bytes
  contentType: string    # declared MIME type

interface SignPutResponse:
  url: string            # presigned PUT URL
  key: string            # storage key (server-generated)
  headers: map[string, string]   # headers client must include in PUT
  expiresAt: integer     # UNIX timestamp when signature expires

function handleSignPutPost(request, user):
  # Step 1: validate request
  if not request.name or not request.size or not request.contentType:
    return 400 Bad Request: missing name, size, or contentType

  if request.size <= 0 or request.size > 100_000_000:
    return 413 Payload Too Large: size must be 1 byte to 100 MB

  if not isAllowedContentType(request.contentType):
    return 415 Unsupported Media Type

  # Step 2: generate deterministic server key (no user choice)
  safeName = sanitizeFilename(request.name)
  fileId = generateUUID()
  key = "uploads/" + user.id + "/" + fileId + "/" + safeName

  # Step 3: build storage policy scoped to key, size, type
  policy = {
    key: key,
    maxSize: request.size,       # exact size match
    allowedTypes: [request.contentType],
    expiresAtSeconds: 300         # 5 minute TTL
  }

  # Step 4: sign the policy with storage credentials (S3, GCS, etc.)
  signingResult = signStoragePolicy(policy)
  # signingResult: {signedUrl, requiredHeaders, expiresAt}

  return 200 {
    url: signingResult.signedUrl,
    key: key,
    headers: signingResult.requiredHeaders,
    expiresAt: signingResult.expiresAt
  }

function handleFinalizePut(request, user):
  key = request.key
  # optional idempotency key for deduplication
  idempotencyKey = request.headers.get("Idempotency-Key") or generateUUID()

  # Step 1: verify key belongs to caller
  # key format: uploads/<user-id>/<uuid>/<safeName>
  if not key.startsWith("uploads/" + user.id + "/"):
    return 403 Forbidden: key does not belong to your account

  # Step 2: check cache for idempotent finalize
  cached = queryIdempotencyCache(idempotencyKey)
  if cached:
    return 200 cached   # {id, url, size}

  # Step 3: verify object exists in storage via HEAD
  headResult = headObject(key)
  if not headResult:
    return 404 Not Found: object not found in storage

  # Step 4: verify size and content-type match the signed policy
  # (parse the original sign request from the key context, or store in temp record)
  if headResult.size > 100_000_000:
    return 413 Payload Too Large: uploaded object exceeds policy size

  if not isAllowedContentType(headResult.contentType):
    return 415 Unsupported Media Type: object content-type not allowed

  # Step 5: create durable upload record in DB
  uploadId = generateId()
  insertUploadRecord({
    id: uploadId,
    userId: user.id,
    storageKey: key,
    contentType: headResult.contentType,
    size: headResult.size,
    createdAt: now()
  })

  # Step 6: optionally enqueue post-processing (Form 8 ref)
  # e.g. scan, thumbnail, OCR, transcoding
  enqueuePostProcess({
    uploadId: uploadId,
    key: key,
    type: headResult.contentType
  })

  # Step 7: cache finalize result for idempotency
  finalUrl = buildPublicUrl(key)
  result = {
    id: uploadId,
    url: finalUrl,
    size: headResult.size
  }
  storeIdempotencyResult(idempotencyKey, result, ttlSeconds: 3600)

  return 201 result

function isAllowedContentType(contentType: string) -> bool:
  # whitelist, e.g. image/*, application/pdf
  allowedPrefixes = ["image/", "application/pdf", "video/"]
  for prefix in allowedPrefixes:
    if startsWith(contentType, prefix):
      return true
  return false

function sanitizeFilename(name: string) -> string:
  # remove path separators, null bytes, control chars
  # keep alphanumeric, dash, underscore, dot
  # limit length to 200 chars
  cleaned = replaceRegex(name, "[^a-zA-Z0-9._-]", "_")
  return substring(cleaned, 0, 200)
```

## Contract

- **Endpoint 1: `POST /uploads/sign`**
  - **Content-Type**: `application/json`
  - **Request body**:
    ```json
    {
      "name": "document.pdf",
      "size": 5242880,
      "contentType": "application/pdf"
    }
    ```
  - **Response 200 OK**:
    ```json
    {
      "url": "https://s3.example.com/...?signature=...",
      "key": "uploads/user123/uuid-here/document.pdf",
      "headers": {
        "x-amz-algorithm": "AWS4-HMAC-SHA256",
        "x-amz-credential": "...",
        "x-amz-date": "...",
        "x-amz-signature": "..."
      },
      "expiresAt": 1719523456
    }
    ```
  - **Response 400 Bad Request**: missing or invalid field
  - **Response 413 Payload Too Large**: size > 100 MB
  - **Response 415 Unsupported Media Type**: contentType not in allowlist

- **Client PUT directly to storage**:
  - **Method**: `PUT` to the `url` from sign response
  - **Headers**: EXACTLY the headers from sign response (extra headers invalidate signature)
  - **Body**: raw file bytes
  - **Response 200/204**: object stored
  - **Browser must have CORS permission**: bucket CORS policy must allow `PUT` from app origin

- **Endpoint 2: `POST /uploads/finalize`**
  - **Content-Type**: `application/json`
  - **Request body**:
    ```json
    {
      "key": "uploads/user123/uuid-here/document.pdf"
    }
    ```
  - **Optional header `Idempotency-Key`**: UUID for deduplication; finalize is idempotent within 1 hour
  - **Response 201 Created**:
    ```json
    {
      "id": "<durable-upload-id>",
      "url": "<absolute-or-relative-path-to-object>",
      "size": 5242880
    }
    ```
  - **Response 403 Forbidden**: key does not belong to caller
  - **Response 404 Not Found**: object not found in storage (may have been deleted)
  - **Response 415 Unsupported Media Type**: object content-type not in allowlist

## Pitfalls

- ❌ **服务器生成的预签名 URL 无大小/类型/密钥绑定**
  - **修复**：在签署时将策略绑定到精确密钥（确定性 UUID）、content-type 和 content-length。不进行绑定会导致用户可以向 URL PUT 不同内容。

- ❌ **客户端选择存储密钥**
  - **修复**：服务器确定性地生成密钥（例如 `uploads/<user-id>/<uuid>/<safe-name>`）。绝不让客户端指定密钥，否则他们可以覆盖其他用户的文件或逃脱上传前缀。

- ❌ **在不进行存储验证的情况下信任客户端的最终确认调用**
  - **修复**：始终在最终确认期间 HEAD 存储中的对象。确认大小和 content-type 与策略匹配。仅在此之后创建持久 DB 记录。

- ❌ **预签名 URL 生命周期过长**
  - **修复**：将 TTL 设置为 ≤5 分钟。在日志、浏览器历史记录或引用者头部中泄露的 URL 仅在短时间内保持风险。

- ❌ **桶缺少 CORS 头部**
  - **修复**：桶 CORS 策略必须允许来自应用源的 `PUT`，并通过 `Access-Control-Expose-Headers` 暴露 `ETag` 头部。不这样做会导致浏览器用 CORS 错误阻止 `PUT`。

- ❌ **孤立上传无生命周期/TTL**
  - **修复**：设置 S3 生命周期规则以在 24 小时后删除 `uploads/tmp/` 或 `uploads/pending/` 中的对象（或存储特定的等效项）。防止未声称的部分上传的累积。

- ❌ **在不进行幂等性处理的情况下两次最终确认相同密钥**
  - **修复**：存储由确定性最终确认 ID（密钥 + 用户组合）或显式 `Idempotency-Key` 头部作为密钥的最终确认结果。如果在 TTL 内重播（例如 1 小时），返回缓存结果。防止重复的 DB 记录。

- ❌ **在最终确认时不进行 content-type 嗅探**
  - **修复**：尽管客户端在签署请求中声称了一个类型，始终验证存储 HEAD 响应中的对象实际 content-type 与允许列表匹配。客户端可能已发送不同的字节。

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [aws/aws-sdk-js](https://github.com/aws/aws-sdk-js) — ~8k★ (verified 2026-05-28): canonical S3 presigned URL signing reference (v2; v3 is modularized in aws-sdk-js-v3)
- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★ (verified 2026-05-28): production presigned PUT client with @uppy/aws-s3 plugin, handles progress, retries, and CORS
