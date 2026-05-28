---
form: multipart_parallel_parts
topic: upload
applies_to: [frontend, backend]
decision: large files (>100MB) needing parallel parts and abort capability
status: stable
last_reviewed: 2026-05-28
---

# Upload 3: Multipart Parallel Parts

客户端按配置的分块大小将文件拆分成多个部分，通过有界并发上传器将每个部分并行上传到存储，
服务器协调初始化、签名生成和最终完成。适合超过 100 MB 的文件、需要并行吞吐量以及中途可中止
（带清理）的场景。服务端向客户端返回可空 MPU（multipart upload）ID 和单个部分签名 URL；客户端
管理有界并发（通常 3–6 个工作线程）、聚合 ETag，最终由服务端调用存储的完成操作。始终提供中止
端点以清理存储侧（否则未完成部分继续被计费）。

## When to use

- 文件 >100 MB：对于直接 PUT（表单 2）过大，对于服务代理（表单 1）不可行
- 需要并行吞吐量：网络延迟不是主要瓶颈，而是端到端和（sum of）吞吐量
- 需要可中止的上传：允许用户在进行中取消并清理存储侧状态
- 存储层暴露多部分上传原始程序（S3 CreateMultipartUpload、GCS 可恢复上传、MinIO）
- 跨会话恢复**不需要**（如需，使用表单 4 ——TUS 或 resumable）

## When NOT to use

- 文件 ≤100 MB：使用表单 2（预签名直接 PUT）足够，更简单
- 弱网环境需要恢复持久性：使用表单 4（TUS / 可恢复）；本形式中止 = 从零重来
- 存储无多部分上传基本型：使用表单 2（直接 PUT）或表单 1（代理）
- 需要跨会话断点续传：表单 4；本形式中止后无法恢复已上传的部分

## Conclusion

三阶段编排——(1) `initiate` 服务器创建多部分上传，返回 uploadId 与建议的分块大小；
(2) `uploadParts` 客户端按 uploadId 逐个请求单部分签名 URL，通过有界并发工作池上传，
收集 `{partNumber, eTag}`；(3) `complete` 服务器调用存储的 `CompleteMultipartUpload`
给定部分列表。始终提供 `abort` 端点清理存储侧的未完成状态（否则库存费用），可选由客户端主动调用或
由桶生命周期规则在 24 小时未完成后自动清理。

## Frontend

```
interface MultipartUploadState:
  idle | initiating | uploading | completing | done | aborted | error

interface UploadPart:
  partNumber: integer
  eTag: string           # from PUT response header
  progress: float        # 0.0 to 1.0

function MultipartFileUpload(props):
  state file: optional Blob = null
  state uploadState: MultipartUploadState = "idle"
  state uploadId: optional string = null
  state parts: map[integer, UploadPart] = {}
  state totalProgress: float = 0.0
  state errorMsg: optional string = null
  state partSize: integer = 5_242_880    # 5 MB default

  # Worker pool to limit concurrency
  state concurrencyLimit: integer = 4
  state inFlightParts: set[integer] = {}

  function onFileSelect(f: Blob):
    if f.size > 5_000_000_000:    # 5 GB is S3 max object size
      errorMsg = "File exceeds 5 GB limit"
      uploadState = "error"
      return
    file = f
    uploadState = "idle"
    errorMsg = null
    uploadId = null
    parts = {}
    initiateUpload(f)

  function initiateUpload(f: Blob):
    uploadState = "initiating"
    errorMsg = null

    try:
      response = await httpRequest({
        method: "POST",
        url: "/uploads/mp/init",
        headers: {
          "Content-Type": "application/json"
        },
        body: {
          key: generateDeterministicKey(f.name),
          contentType: f.type or "application/octet-stream",
          size: f.size
        }
      })

      if response.status != 200:
        throw Error("Failed to initiate multipart upload: " + response.statusText)

      initResponse = response.json()
      # initResponse: {uploadId, partSize, maxParts}

      uploadId = initResponse.uploadId
      partSize = initResponse.partSize

      # Compute part count
      partCount = ceil(f.size / partSize)
      if partCount > initResponse.maxParts:
        throw Error("File requires " + partCount + " parts; max " + initResponse.maxParts)

      uploadState = "uploading"
      uploadParts(f, partCount)

    catch e:
      uploadState = "error"
      errorMsg = e.message
      # Clean up on init failure
      if uploadId:
        abortUpload()

  function uploadParts(f: Blob, partCount: integer):
    # Initialize part tracking
    for i in 1..partCount:
      parts[i] = {partNumber: i, eTag: null, progress: 0.0}

    # Bounded worker pool pattern: run up to concurrencyLimit uploads concurrently
    workers = pool of size concurrencyLimit
    for partNumber in 1..partCount:
      await workers.acquire()
      spawn uploadPartAsync(f, partNumber):
        try:
          # Request signed URL for this part
          signResponse = await httpRequest({
            method: "POST",
            url: "/uploads/mp/part",
            body: {uploadId: uploadId, partNumber: partNumber}
          })
          if signResponse.status != 200:
            throw Error("Failed to sign part " + partNumber)
          partUrl = signResponse.json()

          # Extract and PUT part bytes
          startByte = (partNumber - 1) * partSize
          endByte = min(startByte + partSize, f.size)
          partBlob = f.slice(startByte, endByte)
          putResponse = await httpRequest({
            method: "PUT",
            url: partUrl.url,
            headers: partUrl.headers,
            body: partBlob,
            onProgress: (loaded, total) => {
              parts[partNumber].progress = loaded / total
              updateTotalProgress()
            }
          })

          if putResponse.status != 200 and putResponse.status != 204:
            throw Error("PUT failed for part " + partNumber)

          # Capture ETag from response (required for complete)
          eTag = putResponse.headers.get("ETag")
          if not eTag:
            throw Error("No ETag in PUT response for part " + partNumber)
          parts[partNumber].eTag = eTag
          parts[partNumber].progress = 1.0
          updateTotalProgress()

        catch e:
          parts[partNumber].eTag = null
          errorMsg = "Part " + partNumber + " upload failed: " + e.message
          uploadState = "error"
          abortUpload()
        finally:
          workers.release()

    await workers.allDone()
    completeUpload()

  function updateTotalProgress():
    successBytes = 0
    totalBytes = 0
    for partNum, part in parts:
      startByte = (partNum - 1) * partSize
      endByte = min(startByte + partSize, file.size)
      partBytes = endByte - startByte
      totalBytes += partBytes
      if part.progress > 0.0:
        successBytes += partBytes * part.progress
    totalProgress = successBytes / totalBytes if totalBytes > 0 else 0.0

  function completeUpload():
    if uploadState != "uploading":
      return

    # Verify all parts have ETags
    for partNum in 1..parts.size():
      if not parts[partNum].eTag:
        uploadState = "error"
        errorMsg = "Part " + partNum + " missing ETag; cannot complete"
        abortUpload()
        return

    uploadState = "completing"

    try:
      partsList = []
      for partNum, part in parts:
        partsList.append({partNumber: partNum, eTag: part.eTag})

      response = await httpRequest({
        method: "POST",
        url: "/uploads/mp/complete",
        headers: {
          "Content-Type": "application/json"
        },
        body: {
          uploadId: uploadId,
          parts: partsList
        }
      })

      if response.status != 200 and response.status != 201:
        throw Error("Complete failed: " + response.statusText)

      result = response.json()
      uploadState = "done"
      totalProgress = 1.0
      props.onSuccess(result)

    catch e:
      uploadState = "error"
      errorMsg = e.message
      # Server-side complete failed; abort to clean up
      abortUpload()

  function abortUpload():
    if not uploadId:
      return

    try:
      await httpRequest({
        method: "POST",
        url: "/uploads/mp/abort",
        headers: {
          "Content-Type": "application/json"
        },
        body: {
          uploadId: uploadId
        }
      })
    catch e:
      # Log but don't fail; storage will clean up via lifecycle
      console.warn("Abort request failed; storage will clean up: " + e.message)

  function cancel():
    if uploadState == "uploading" or uploadState == "initiating":
      abortUpload()
      uploadState = "aborted"
      totalProgress = 0.0
      uploadId = null
      parts = {}
      inFlightParts = {}

  render:
    if uploadState == "idle":
      file-input (accept any, validate size on select)
    if uploadState == "initiating":
      spinner ("Preparing multipart upload...")
    if uploadState == "uploading":
      progress-bar (totalProgress)
      part-breakdown (map of parts with individual progress)
      cancel-button
      concurrency-control (optional slider 1–8)
    if uploadState == "completing":
      spinner ("Finalizing upload...")
    if uploadState == "error":
      error-message (errorMsg)
      retry-button (calls initiateUpload(file))
      abort-button (calls abortUpload and close)
    if uploadState == "aborted":
      message ("Upload cancelled and cleaned up")
    if uploadState == "done":
      success-checkmark
      result (id, url, size)
```

## Backend

```
interface InitiateRequest:
  key: string            # server-generated or deterministic client key
  contentType: string    # declared MIME type
  size: integer          # total file size

interface InitiateResponse:
  uploadId: string       # storage multipart upload ID
  partSize: integer      # server-recommended or storage default
  maxParts: integer      # storage max parts (e.g. 10000 for S3)

function handleMpInitPost(request, user):
  # Step 1: validate request
  if not request.key or not request.size or not request.contentType:
    return 400 Bad Request

  if request.size <= 0 or request.size > 5_000_000_000:
    return 413 Payload Too Large

  if not isAllowedContentType(request.contentType):
    return 415 Unsupported Media Type

  # Step 2: ensure key belongs to user (deterministic prefix)
  # Example: uploads/<user-id>/<uuid>/<safe-name>
  if not request.key.startsWith("uploads/" + user.id + "/"):
    return 403 Forbidden

  # Step 3: compute part size (5 MB to 5 GB, divisor ≤ 10,000 parts)
  MIN_PART_SIZE = 5_242_880      # 5 MB
  MAX_PART_SIZE = 5_368_709_120  # 5 GB
  MAX_PARTS = 10000

  partSize = max(MIN_PART_SIZE, ceil(request.size / 9999))
  partSize = min(partSize, MAX_PART_SIZE)
  partCount = ceil(request.size / partSize)

  if partCount > MAX_PARTS:
    return 413 Payload Too Large: "File requires " + partCount + " parts; max 10,000"

  # Step 4: create multipart upload and store session
  mpu = createMultipartUpload({bucket: UPLOAD_BUCKET, key: request.key, contentType: request.contentType})
  storeUploadSession({uploadId: mpu.uploadId, userId: user.id, key: request.key, contentType: request.contentType, totalSize: request.size, partSize: partSize, expiresAt: now() + 86400})
  return 200 {uploadId: mpu.uploadId, partSize: partSize, maxParts: MAX_PARTS}

interface SignPartRequest:
  uploadId: string
  partNumber: integer

interface SignPartResponse:
  url: string            # presigned PUT URL for this part
  headers: map[string, string]

function handleMpPartPost(request, user):
  # Step 1: validate request
  if not request.uploadId or not request.partNumber:
    return 400 Bad Request

  if request.partNumber < 1 or request.partNumber > 10000:
    return 400 Bad Request: "partNumber out of range"

  # Step 2: retrieve upload session
  session = queryUploadSession(request.uploadId)
  if not session:
    return 404 Not Found: "Upload session not found"

  # Step 3: verify ownership
  if session.userId != user.id:
    return 403 Forbidden

  # Step 4: generate signed PUT URL for this part
  policy = {
    bucket: UPLOAD_BUCKET,
    key: session.key,
    uploadId: request.uploadId,
    partNumber: request.partNumber,
    expiresAtSeconds: 300    # 5 minute TTL per part
  }

  try:
    signResult = signStoragePart(policy)
    # signResult: {url, headers}

  catch e:
    return 500 Internal Server Error

  return 200 {
    url: signResult.url,
    headers: signResult.headers
  }

interface CompleteRequest:
  uploadId: string
  parts: array[{partNumber: integer, eTag: string}]

interface CompleteResponse:
  id: string             # durable upload record ID
  url: string            # final accessible path
  size: integer          # verified total size
  key: string            # storage key

function handleMpCompletePost(request, user):
  # Step 1: validate request
  if not request.uploadId or not request.parts or request.parts.isEmpty():
    return 400 Bad Request

  # Step 2: retrieve and validate session
  session = queryUploadSession(request.uploadId)
  if not session:
    return 404 Not Found

  if session.userId != user.id:
    return 403 Forbidden

  # Step 3: sort parts by partNumber and verify contiguity
  sortedParts = sort(request.parts, by: partNumber)
  for i, part in sortedParts:
    if part.partNumber != i + 1:
      return 400 Bad Request: "Parts not contiguous"

  # Step 4: build part list for storage CompleteMultipartUpload
  storagePartList = []
  for part in sortedParts:
    storagePartList.append({
      PartNumber: part.partNumber,
      ETag: part.eTag
    })

  # Step 5: invoke storage complete and verify size
  result = completeMultipartUpload({bucket: UPLOAD_BUCKET, key: session.key, uploadId: request.uploadId, parts: storagePartList})
  if result.size != session.totalSize:
    return 400 Bad Request: "Size mismatch: expected " + session.totalSize + ", got " + result.size

  # Step 6: create durable upload record and clean up session
  uploadId = generateId()
  insertUploadRecord({id: uploadId, userId: user.id, storageKey: session.key, contentType: session.contentType, size: result.size, createdAt: now()})
  deleteUploadSession(request.uploadId)
  enqueuePostProcess({uploadId: uploadId, key: session.key, type: session.contentType})
  return 201 {
    id: uploadId,
    url: buildPublicUrl(session.key),
    size: result.size,
    key: session.key
  }

interface AbortRequest:
  uploadId: string

function handleMpAbortPost(request, user):
  # Step 1: retrieve session
  session = queryUploadSession(request.uploadId)
  if not session:
    return 404 Not Found

  # Step 2: verify ownership
  if session.userId != user.id:
    return 403 Forbidden

  # Step 3: invoke storage abort
  try:
    abortMultipartUpload({
      bucket: UPLOAD_BUCKET,
      key: session.key,
      uploadId: request.uploadId
    })

  catch e:
    if "NoSuchUpload" in e.message:
      # Already aborted or expired; ignore
      pass
    else:
      return 500 Internal Server Error

  # Step 4: clean up session
  deleteUploadSession(request.uploadId)

  return 204 No Content
```

## Contract

- **Endpoint 1: `POST /uploads/mp/init`**
  - **Request**:
    ```json
    {
      "key": "uploads/user123/uuid/document.pdf",
      "contentType": "application/pdf",
      "size": 157286400
    }
    ```
  - **Response 200 OK**:
    ```json
    {
      "uploadId": "abc-123-def-456",
      "partSize": 5242880,
      "maxParts": 10000
    }
    ```
  - **Response 400 Bad Request**: missing fields or invalid data
  - **Response 413 Payload Too Large**: size > 5 GB or requires > 10,000 parts
  - **Response 415 Unsupported Media Type**: contentType not allowed

- **Endpoint 2: `POST /uploads/mp/part`**
  - **Request**:
    ```json
    {
      "uploadId": "abc-123-def-456",
      "partNumber": 5
    }
    ```
  - **Response 200 OK**:
    ```json
    {
      "url": "https://s3.example.com/...?...",
      "headers": {
        "x-amz-algorithm": "...",
        "x-amz-credential": "...",
        "x-amz-date": "...",
        "x-amz-signature": "..."
      }
    }
    ```
  - **Response 400 Bad Request**: invalid partNumber
  - **Response 404 Not Found**: uploadId not found

- **Client PUT to signed part URL**:
  - **Method**: `PUT` to `url` with `headers` from sign response
  - **Body**: raw bytes for this part (5 MB to 5 GB)
  - **Response 200/204**: part accepted
  - **Client must capture `ETag` header** from response; required for complete

- **Endpoint 3: `POST /uploads/mp/complete`**
  - **Request**:
    ```json
    {
      "uploadId": "abc-123-def-456",
      "parts": [
        {"partNumber": 1, "eTag": "\"abc123def456\""},
        {"partNumber": 2, "eTag": "\"xyz789uvw012\""}
      ]
    }
    ```
  - **Response 201 Created**:
    ```json
    {
      "id": "upload-final-id",
      "url": "https://example.com/files/document.pdf",
      "size": 157286400,
      "key": "uploads/user123/uuid/document.pdf"
    }
    ```
  - **Response 400 Bad Request**: parts missing, not contiguous, or malformed
  - **Response 404 Not Found**: uploadId not found
  - **Response 500 Internal Server Error**: storage complete failed

- **Endpoint 4: `POST /uploads/mp/abort`**
  - **Request**:
    ```json
    {
      "uploadId": "abc-123-def-456"
    }
    ```
  - **Response 204 No Content**: abort successful (idempotent)
  - **Response 404 Not Found**: uploadId not found

- **Part size range**: 5 MB–5 GB per part; max 10,000 parts per upload (S3 spec)
- **ETag persistence**: client must store and echo all part ETags at complete (immutable once PUT)

## Pitfalls

- ❌ **Unbounded concurrent part uploads → connection storms or link failures**
  - **Fix**: Implement bounded worker pool with 3–6 concurrent uploaders; limit simultaneous PUT requests.

- ❌ **Omit `abort` cleanup → storage continues billing incomplete multipart uploads**
  - **Fix**: Client calls `abort` on user cancel; bucket lifecycle rule auto-cleans incomplete uploads in `uploads/` prefix after 24 hours.

- ❌ **Part size <5 MB (S3 minimum, except last part) → complete fails**
  - **Fix**: Compute `partSize = max(5MB, ceil(totalSize / 9999))`; never hardcode small values like 1 MB.

- ❌ **Re-sign entire upload on each retry → unnecessary signing round-trips**
  - **Fix**: Make signing idempotent by `(uploadId, partNumber)`; cache result 5 minutes. Retry same part reuses same signed URL.

- ❌ **Trust client part list at complete without verification → storage may assemble parts from different users**
  - **Fix**: Server verifies assembled size matches declared; verify part count is contiguous (1..N).

- ❌ **Long-running uploads exceed signed URL TTL → PUT rejected or accepts invalid signature**
  - **Fix**: Set single-part TTL ≥ expected PUT duration. For parts needing >5 min, provide lazy refresh (client requests fresh signed URL as needed).

- ❌ **Unbounded partNumber or skipped validation at init → client uploads with invalid uploadId**
  - **Fix**: Init endpoint validates size in range, part count ≤ 10,000, contentType allowed; reject invalid requests.

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★ (verified 2026-05-28): production-grade parallel-parts S3 client with @uppy/aws-s3-multipart plugin, handles concurrency, ETags, and abort lifecycle
- [aws/aws-sdk-js](https://github.com/aws/aws-sdk-js) — ~8k★ (verified 2026-05-28): canonical S3 multipart upload server-side API reference and client SDK
