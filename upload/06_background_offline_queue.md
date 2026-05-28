---
form: background_offline_queue
topic: upload
applies_to: [frontend]
decision: PWA / offline-first / uploads must survive tab close and flaky mobile networks
status: stable
last_reviewed: 2026-05-28
---

# Upload 6: Background / Offline Upload Queue

PWA 和离线优先应用中，用户选择文件后立即关闭标签页或遭遇网络中断，上传不应丢失。本 form 将待传上传持久化到 IndexedDB（文件 blob + 状态机），注册 Service Worker Background Sync API，连接恢复时由 SW 后台重放队列。UI 线程与上传解耦——关 tab 不杀上传（在 SW 生命周期内）。与 form 04 (tus) 组合可在 SW 本身被杀时通过 Idempotency-Key 恢复。

## When to use

- PWA 或移动现场应用：用户预期离线后能继续上传
- Fire-and-forget 场景：选完文件马上关 tab，后台继续传
- 弱网或移动网络：频繁中断，需要 SW 自主重试而非页面驱动
- 与 form 04 (tus) 组合：双重 resilient——既有 tus 恢复，又有 SW 队列备份
- 用户体验：无需轮询"上传成功了吗"，通知告知最终状态

## When NOT to use

- 桌面 Web 应用，需强调页内实时进度条：用户需看到 0–100% 的可视进度
- 无 Service Worker 支持的环境（旧浏览器或 HTTP-only 非 HTTPS 站点）
- 单个文件 < 5 MB 且网络稳定：form 01/02 足够
- 用户不接受后台处理：规范 UX 需明确告知"关 tab 后仍在上传"

## Conclusion

将待传上传的 (blob, metadata, state) 三元组持久化到 IndexedDB，每条记录带稳定的 idempotency key。UI 层次通过 Form 01–04 任一方式发起上传时，同时写入队列；成功后清除队列记录；临时失败时 backoff 重试；永久失败（4xx）则标记 `dead` 让用户驱动补救。Service Worker 在后台监听 `sync` 事件，迭代队列，调用底层传输（01–04），成功则 `idb.delete(id)` 并通过 `postMessage` / `BroadcastChannel` 通知前台；失败则 backoff 并重新注册 sync。状态机流转：`pending → uploading → succeeded | dead`，UI 可通过 `BroadcastChannel` 订阅进度。

## Frontend

```
interface QueuedUpload:
  id: string                    # UUID，enqueue 时生成，持久化为 idempotency key
  blob: Blob
  targetPath: string            # "/api/uploads" 或其他端点
  originalName: string
  mimeType: string
  state: string                 # pending | uploading | succeeded | dead
  attempts: integer             # 重试计数
  errorMsg: optional string
  timestamp: integer            # enqueue 时刻（unix ms）
  idempotencyKey: string        # 同 id；供后端幂等校验

# IndexedDB schema
database: upload_queue
  store: uploads
    keyPath: id
    indexes: [state, timestamp]

async function enqueueUpload(blob: Blob, targetPath: string, originalName: string) -> string:
  # 主线程立即入队，不等网络
  db = await openIndexedDB("upload_queue")
  
  id = generateUUID()
  record: QueuedUpload = {
    id: id,
    blob: blob,
    targetPath: targetPath,
    originalName: originalName,
    mimeType: blob.type or "application/octet-stream",
    state: "pending",
    attempts: 0,
    errorMsg: null,
    timestamp: Date.now(),
    idempotencyKey: id
  }
  
  await db.put("uploads", record)
  
  # 注册 SW Background Sync（若支持）
  if navigator.serviceWorker and "sync" in registration:
    await registration.sync.register("replay-queue")
  
  # 通知前台订阅者（若任何 UI 在听）
  broadcastToClients({type: "queued", id: id, name: originalName})
  
  return id

# Service Worker 侧
self.addEventListener("sync", event => {
  if event.tag == "replay-queue":
    event.waitUntil(replayUploadQueue())
})

async function replayUploadQueue():
  db = await openIndexedDB("upload_queue")
  
  # 取出所有 pending 和 uploading 状态的记录
  records = await db.getAll("uploads", 
    IDBKeyRange.bound(["pending"], ["uploading"]))
  
  for record in records:
    success = await attemptUpload(record)
    if success:
      await db.delete("uploads", record.id)
      broadcastToClients({
        type: "upload-success",
        id: record.id,
        name: record.originalName
      })
    else:
      # 临时失败（网络错误）：backoff + 重新注册 sync
      if record.attempts < 10:
        record.attempts = record.attempts + 1
        record.state = "pending"  # 回到 pending，等待下次 sync
        await db.put("uploads", record)
        
        # 指数退避重试：30s + (2^attempts - 1) 倍
        backoffMs = 30000 * (pow(2, record.attempts) - 1)
        await sleep(backoffMs)
        await registration.sync.register("replay-queue")
      else:
        # 已重试 10 次，标记为死队列，让用户决定
        record.state = "dead"
        record.errorMsg = "Max retries exceeded"
        await db.put("uploads", record)
        broadcastToClients({
          type: "upload-dead",
          id: record.id,
          error: record.errorMsg
        })

async function attemptUpload(record: QueuedUpload) -> boolean:
  try:
    # 调用底层传输（Form 01–04 之一）
    formData = new FormData()
    formData.append("file", record.blob)
    formData.append("name", record.originalName)
    formData.append("Idempotency-Key", record.idempotencyKey)
    
    response = await fetch(record.targetPath, {
      method: "POST",
      body: formData,
      headers: {
        "Idempotency-Key": record.idempotencyKey
      }
    })
    
    if response.ok:
      record.state = "succeeded"
      return true
    
    if response.status >= 400 and response.status < 500:
      # 4xx：永久错误（文件损坏、权限不足等）
      record.state = "dead"
      record.errorMsg = "Server error: " + response.statusText
      await db.put("uploads", record)
      return false
    
    # 5xx 或网络错误：临时错误，应该重试
    return false
  
  catch error:
    # 网络错误、超时等
    return false

# 前台页面订阅队列状态
function subscribeToUploadQueue(onUpdate):
  if not window.BroadcastChannel:
    console.warn("BroadcastChannel not supported")
    return
  
  channel = new BroadcastChannel("upload-queue")
  channel.onmessage = (event) => {
    msg = event.data
    if msg.type in ["queued", "upload-success", "upload-dead"]:
      onUpdate(msg)
  }
  
  return () => channel.close()

# UI 集成示例
function FileUploadWithQueue(props):
  state queuedUploads: Map<string, QueuedUpload> = new Map()
  
  function onFileSelect(file: Blob):
    try:
      id = await enqueueUpload(file, "/api/uploads", file.name)
      queuedUploads.set(id, {
        id: id,
        originalName: file.name,
        state: "pending"
      })
      showToast("Queued: " + file.name)
    catch e:
      showError("Failed to queue: " + e.message)
  
  function setupQueueListener():
    unsubscribe = subscribeToUploadQueue((msg) => {
      if msg.type == "queued":
        queuedUploads.set(msg.id, {id: msg.id, originalName: msg.name, state: "pending"})
      elif msg.type == "upload-success":
        queuedUploads.delete(msg.id)
        showNotification("Upload complete: " + msg.name)
      elif msg.type == "upload-dead":
        record = queuedUploads.get(msg.id)
        if record:
          record.state = "dead"
          record.errorMsg = msg.error
    })
    return unsubscribe
  
  onMount:
    return setupQueueListener()
  
  render:
    file-input (onFileSelect)
    if queuedUploads.size > 0:
      queue-list:
        for each id, record in queuedUploads:
          queue-item:
            - name: record.originalName
            - state: record.state (badge color: pending=gray, uploading=blue, dead=red)
            - if dead: retry-button (onClick: requeue(id))
```

## Backend

本 form 无独立后端协议。后端 MUST 在 form 01–04 的 finalize 处实现**幂等性**（SW 可能因为响应丢失而重放已成功上传）。每个排队上传必须带 `Idempotency-Key` header；后端据此判断"这是新上传还是重试"——新上传则存储、重试则返回先前的成功响应（无副作用）。详见 form 02/03/04 的 Backend 节及 Idempotency-Key 约定。

## Contract

继承所选传输 form (01–04) 的 contract。补充：

- **POST /api/uploads**（或其他路由）
  - Header：`Idempotency-Key: <id>`（UUID，每次上传固定）
  - Body：multipart FormData with `file`, `name`, 其他元数据
  - Response 201：`{id, url, size}` — 同一 Idempotency-Key 重复请求返回**相同响应**，无副作用
  - Errors：400 (invalid file), 413 (too large), 409 (quota exceeded)

后端必须验证：
1. Idempotency-Key 格式有效（UUID）
2. 同一 key 的重试返回先前成功响应（检查 idempotency cache）
3. 新 key 则正常存储（form 01–04 的逻辑）

## Pitfalls

- ❌ **blob 仅在内存中持有**：tab 关闭时丢失。
  - **Fix**：enqueue 时将 blob 持久化到 IndexedDB。

- ❌ **无幂等性密钥**：重放时重复创建资源。
  - **Fix**：enqueue 时生成稳定 UUID，同时用作 `Idempotency-Key` header。

- ❌ **队列无限增长**：死上传堆积，浪费存储。
  - **Fix**：attempts 上限（如 10）→ `state='dead'`；用户驱动重试或清除。

- ❌ **仅通过 UI 线程通知**：用户已关闭 tab 时，前台卸载导致状态丢失。
  - **Fix**：SW 通过 `showNotification()` 在终态（成功/失败）通知用户；前台重新打开时通过 BroadcastChannel 同步队列状态。

- ❌ **SW 存储配额耗尽**：嵌入 blob 且不清理会堆满 IndexedDB。
  - **Fix**：成功上传立刻 `db.delete(id)`；`navigator.storage.estimate()` 预检查——剩余 < 2× 文件大小则拒绝 enqueue。

- ❌ **重试无上限或无指数退避**：泛滥 API、破坏服务。
  - **Fix**：最多 10 次重试，每次 backoff = 30s × (2^attempt - 1)；超过 10 次改为 dead 需人工介入。

- ❌ **前台未订阅 BroadcastChannel**：SW 完成上传却没人听，用户不知道。
  - **Fix**：前台挂载时立刻 `subscribeToUploadQueue()`，保持频道打开（直到卸载）。

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [GoogleChrome/workbox](https://github.com/GoogleChrome/workbox) — ~13k★ (verified 2026-05-28)：生产级 Service Worker 工具库，workbox-background-sync 模块提供队列持久化、自动重试、指数退避、幂等注册
- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★ (verified 2026-05-28)：生产级上传 UI 组件，Golden Retriever 插件实现跨 tab 持久化与恢复（IndexedDB 存储、自动重放）
