---
form: post_upload_pipeline
topic: upload
applies_to: [backend]
decision: derive artifacts (thumbnails, transcodes, scans) after upload via event-driven jobs
status: stable
last_reviewed: 2026-05-28
---

# Upload 8: Post-Upload Processing Pipeline

上传只是入口，真正业务是派生（缩略图、转码、OCR、扫毒、CDN 预热）。想要解耦上传延迟与处理时间，避免客户端长时间等待。对象存储在 `PutObject` finalize 时 emit 事件，fan out 到多个 worker；每个 worker 幂等处理派生物，写入独立 prefix；client 轮询或订阅状态直到就绪。

## When to use

- 派生物处理时间不可预测或超过 500 ms（缩略图、转码、OCR、病毒深度扫）
- 多个派生物并发生成，想要独立伸缩（image worker 和 video worker 分离）
- 上传完成与派生完成需要异步分离，client 不等待全部完成
- 派生物失败不应阻断原始上传成功确认
- 业务需要重试失败的派生任务且无法在上传时同步决策

## When NOT to use

- 单个派生 200 ms 内完成（e.g. 一张缩略图或 md5 hash）— 直接 inline 处理于上传请求内
- 派生物是上传完整性的前置条件（e.g. 必须扫描才能返回成功）— 改为同步，不异步分离
- 派生物数量固定且场景简单（只需 1 转码）— 考虑上传时 pipe 一道转码流而非事件驱动

## Conclusion

对象存储（S3 / GCS / Blob） emit 「object created」事件至消息队列或事件总线（SQS / EventBridge / Pub/Sub），key 为 `uploads/{uploadId}` / `{sha256}` 格式。多个 worker subscribe 该事件：image-derive worker 处理图片，video-transcode worker 处理视频，scan worker 做病毒检查。每个 worker 从事件读取原始对象元数据（key、size、content-type、hash），检查派生物是否已存在（内容寻址 key `derived/{sha256}/{variant}`），若不存在则处理并写入。处理完毕后更新 job-status 表（DynamoDB / RDS 行，key = uploadId），标记对应派生物 ready。Client 轮询 `GET /uploads/{id}` 或 subscribe SSE / WebSocket 状态频道，读取 `status: processing` → 派生物列表 `derivatives: {thumb, web, avif, ...}` → 最终 `status: done`。所有 worker 必须幂等（相同输入 = 相同输出）且 retry-safe；若派生物已存在且 hash 匹配则 skip，避免重复计算。

## Frontend

客户端在 finalize response 后看到 `status: processing`。可选轮询 `GET /uploads/{id}` （推荐 1-2s 间隔）或建立 SSE 连接 `event-stream /uploads/{id}/subscribe` 订阅 `upload.status` 事件。渲染 skeleton / progress bar 直到 `derivatives.thumb_200` 和 `scan_status: clean` ready。public CDN URL 仅从 `derived/*` prefix serve，未扫描的原件 URL 返回 403。

## Backend

```
on CloudEvent "storage.object.finalized":
  uploadId = event.metadata.uploadId
  key = event.key  # "uploads/{uploadId}"
  size = event.size
  contentType = event.contentType
  sha256 = event.sha256 (computed by storage)

  # 发布事件至消息队列 / 事件总线
  publish UploadFinalized{
    uploadId, key, size, contentType, sha256
  } to queue "upload-events"

  # 同步更新 job 状态
  jobs.updateStatus(uploadId, {
    status: "processing",
    derivatives: {}
  })

---

worker "image-derive" subscribing to UploadFinalized:

  on UploadFinalized:
    if contentType does not start with "image/":
      return  # skip non-images

    for variant in [thumb-200, web-1280, avif-1280]:
      derivedKey = f"derived/{sha256}/{variant}"
      
      # 幂等：已存在跳过
      if storage.headObject(derivedKey).exists():
        jobs.updateStatus(uploadId, {
          derivatives: {thumb: derived-url, ...}
        })
        continue
      
      # 下载原始
      bytes = storage.get(key)
      
      # 处理（伪代码）
      output = processImage(bytes, variant)  # sharp / imgproxy call
      
      # 写派生物
      storage.put(derivedKey, output, {
        ContentType: "image/webp" or "image/jpeg",
        CacheControl: "max-age=31536000"  # immutable
      })

    # 全部完成后更新状态
    jobs.updateStatus(uploadId, {
      status: "processing",
      image_status: "ready",
      derivatives: {
        thumb_200: s3Url("derived/{sha256}/thumb-200"),
        web_1280: s3Url("derived/{sha256}/web-1280"),
        avif_1280: s3Url("derived/{sha256}/avif-1280")
      }
    })

---

worker "scan" subscribing to UploadFinalized:

  on UploadFinalized:
    try:
      stream = storage.getStream(key)
      verdict = clamav.scan(stream, timeout: 120_000)  # 2 min
      
      if verdict == "infected":
        storage.delete(key)
        jobs.updateStatus(uploadId, {
          status: "quarantined",
          reason: "infected"
        })
      else:
        jobs.updateStatus(uploadId, {
          scan_status: "clean"
        })
    
    catch error:
      log("scan error", error)
      # 扫描失败不应阻断上传，标记 "scan_pending"
      jobs.updateStatus(uploadId, {
        scan_status: "pending"
      })
      # 重试逻辑由 message queue 的 max attempts 处理

---

function handleGetUploadStatus(uploadId):
  row = jobs.get(uploadId)
  if not row:
    return 404

  return 200 {
    id: uploadId,
    status: row.status,  # "processing" | "done" | "quarantined"
    size: row.size,
    contentType: row.contentType,
    created: row.createdAt,
    derivatives: row.derivatives or {},  # {thumb_200, web_1280, avif_1280, ...}
    scan_status: row.scan_status,  # "clean" | "pending" | "infected"
    publicUrl: (if scan_status == "clean")
                 s3Url("derived/{sha256}/web-1280")
               else
                 null
  }
```

## Contract

- **Job Status Endpoint**: `GET /uploads/{uploadId}` → `{id, status, derivatives, scan_status, publicUrl, ...}`
- **Status Values**: `"processing"` (initial) → `"done"` (all derivatives ready & scan clean) | `"quarantined"` (infected)
- **Derivatives Object**: `{thumb_200: url, web_1280: url, avif_1280: url, scan: "clean" | "pending"}`
- **SSE / WebSocket** (optional): `GET /uploads/{uploadId}/subscribe` → `event: upload.status` data `{...same JSON as GET...}`
- **Worker Retry**: SQS visibility timeout ≥ max processing time；max attempts = 3；fail → dead-letter queue + alert
- **Idempotency**: Derived key = `derived/{sha256}/{variant}`；检查 ETag or `HeadObject`；相同 hash 若已存在，跳过重新处理

## Pitfalls

- ❌ **派生 worker 非幂等，重试导致重复生成**
  - **Fix**: 内容寻址 key `derived/{sha256}/{variant}`；处理前 `headObject()` check；存在即 skip。

- ❌ **上传请求中做长 transcode，客户端 timeout**
  - **Fix**: 上传立即返回 202；transcode via event-driven worker；client 轮询状态。

- ❌ **Job 队列无背压或 worker 并发无限，导致内存爆炸**
  - **Fix**: SQS 或 Pub/Sub 默认背压；worker 限制并发 (e.g. 5 concurrent image jobs)；超载告警。

- ❌ **公开原始 URL 而非扫描后的派生 URL，安全隐患**
  - **Fix**: 原件 key 设 private（IAM deny public）；仅 `derived/*` prefix allow public；contract 返回 `publicUrl` 指向 `derived/{sha256}/web` 而非原件。

- ❌ **扫描 timeout / 失败导致整个上传卡住**
  - **Fix**: 扫描异步后台执行；timeout 则放行但标记 `scan_status: "pending"`；失败不改变 `status: done` 的决策，只影响 `scan_status`。

- ❌ **无 dead-letter queue，卡住的 worker 任务无人问津**
  - **Fix**: SQS DLQ + CloudWatch alarm；人工 inspect DLQ 后重驱动或标记 failed。

- ❌ **多个 worker 并发写同一派生 key，冲突覆盖**
  - **Fix**: 内容寻址 + hash 保证相同源生成相同派生；或加 conditional put (ETag match)；或单 worker per variant。

- ❌ **派生物种类后续扩展困难，worker 硬编码 variant list**
  - **Fix**: Job row 存 `expected_derivatives: [...]`；worker 根据 event metadata 或 config 动态选择。

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [imgproxy/imgproxy](https://github.com/imgproxy/imgproxy) — ~11k★ (verified 2026-05-28): on-the-fly image derivation server; stateless thumbnail & transform worker reference
- [h2non/imaginary](https://github.com/h2non/imaginary) — ~6k★ (verified 2026-05-28): HTTP image processing microservice; async derivative pipeline pattern
- [lovell/sharp](https://github.com/lovell/sharp) — ~32k★ (verified 2026-05-28): embedded image processing for Node.js; reference for idempotent, hash-keyed image transformation workers
