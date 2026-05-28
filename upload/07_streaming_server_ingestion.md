---
form: streaming_server_ingestion
topic: upload
applies_to: [backend]
decision: when proxy is unavoidable, stream don't buffer; bound memory regardless of body size
status: stable
last_reviewed: 2026-05-28
---

# Upload 7: Streaming Server Ingestion

必须代理上传（合规/区域网关/应用层转码），但文件大小无界且不能 OOM。边流边解析，以恒定内存边界处理：多部分 (multipart) 解析器返回每个文件部分的流，经过大小计数、魔数嗅探、病毒扫描等执行阶段，再 pipe 进对象存储上传。背压必须传播：sink 慢则 request 暂停，source 慢则 sink 等待。永不收集到 Buffer。

## When to use

- 文件大小无界（无 5 MB 上限）但必须代理，无法直传对象存储（如合规要求应用层日志/审计）
- 实时边流边转码或边扫描（同步病毒检查、缩略图生成需初始数据流）
- 网关层限制无 presigned URL 或跨区域认证困难
- 并发代理多个大文件，内存池有限，需严格 backpressure
- 区域隔离：上传端和存储端不在同一区域，应用层做 cross-region relay

## When NOT to use

- 允许客户端直传对象存储（form 02/03 更便宜）
- body 保证小（< 5 MB）且无实时转码需求（form 01 简洁高效）
- 无病毒扫描需求且无业务转码（用对象存储 presigned URL 直传）
- 文件单个超过 5 GB 且网络不稳定（form 04 tus resumable 更合适）

## Conclusion

把 HTTP request body 当作 `Readable` 流。pipe 过多部分解析器（emit per-part streams 给每个文件），再 pipe 过执行阶段（计数器检查 max bytes、魔数嗅探器在前 KB 拦截不合法类型、病毒扫描流）。最后 pipe 进 sink（从流构造的对象存储上传）。背压必须传播整条链：对象存储写入慢 → request 暂停读，request 源慢 → 对象存储等待。错误在任何阶段触发时，abort 上传上下文，清理孤儿分片。与 form 01 共用同一 HTTP contract，補充 `Expect: 100-continue` 支持让客户端在 4xx pre-flight 时早退。

## Frontend

与 form 01 客户端协议完全一致（multipart FormData POST `/uploads`）。本 form 是后端对同一 wire 协议的实现差异。详见 [01_server_proxied_multipart.md](01_server_proxied_multipart.md)。

## Backend

```
function handleUploadPost(request):
  # 背压感知管道：不缓冲整个 body，流式处理

  parser = createMultipartStreamParser(request.body)
  
  on parser.part(part):
    # 每个 multipart 部分单独处理，emitted as stream
    
    if part.fieldName != "file":
      # 普通字段快速读取
      part.asString((value) => extraFields[part.fieldName] = value)
      return
    
    # Step 1: 计数器流，超出立即 error
    counter = CountingStream(maxBytes: 500_000_000)  # 500 MB example
    counter.on("exceed", () => {
      abortRequest(request, 413)
      abortStorageUpload()
    })
    
    # Step 2: 魔数嗅探器，前 KB 检验，失败立即 abort source
    sniffer = MagicByteSniffer(allowlist: [
      "image/jpeg", "image/png", "image/gif",
      "application/pdf"
    ])
    sniffer.on("invalid", () => {
      abortRequest(request, 415)
      abortStorageUpload()
    })
    
    # Step 3: 病毒扫描流（可选异步）
    scanner = VirusScanStream({
      mode: "inline",  # 同步小文件；大 > 50MB 改 "quarantine"
      timeout: 30_000   # 30s 超时则放行（游戏规则）
    })
    scanner.on("threat", () => {
      abortRequest(request, 451)  # Unavailable For Legal Reasons
      abortStorageUpload()
    })
    
    # Step 4: 对象存储上传，从 stream 构造
    storageUpload = storage.createMultipartUpload({
      key: deriveStorageKey(part),
      contentType: part.contentType or "application/octet-stream"
    })
    
    # Step 5: 背压链接：part.stream → counter → sniffer → scanner → storageUpload
    part.stream
      .pipe(counter, { end: true })
      .pipe(sniffer, { end: true })
      .pipe(scanner, { end: true })
      .pipe(storageUpload.body, { end: true })
    
    # 最终化
    on storageUpload.finish():
      filePart = {
        id: storageUpload.finalKey,
        url: storageUpload.publicUrl,
        size: counter.totalBytes,
        etag: storageUpload.etag
      }
      return 201 { id, url, size, etag }
    
    on storageUpload.error(error):
      # 上传失败时，abort source（背压传播）
      request.pause()
      counter.destroy()
      sniffer.destroy()
      return 500 Internal Server Error

function CountingStream(maxBytes):
  # 简单计数，超出限制立刻 error
  totalBytes = 0
  
  on _transform(chunk):
    totalBytes += chunk.length
    if totalBytes > maxBytes:
      emit("exceed")
      this.destroy()
    else:
      this.push(chunk)

function MagicByteSniffer(allowlist):
  # 前 1024 字节检验，pass-through
  bufferedBytes = 0
  firstChunk = true
  
  on _transform(chunk):
    if firstChunk:
      firstChunk = false
      sniffedType = detectType(chunk.slice(0, min(1024, chunk.length)))
      if sniffedType not in allowlist:
        emit("invalid")
        this.destroy()
        return
    
    this.push(chunk)

function VirusScanStream(options):
  # 如果 mode == "inline"：传递到 clamav async
  # 大于 50 MB 自动转 "quarantine"（后台扫）
  
  if options.mode == "quarantine":
    # 写到隔离桶，返回 202 Accepted + poll URL
    this.push(chunk)  # stream through
    emit("scanning-async")
  else:
    # inline：等 clamav 扫，超时则放行
    scanAsync(chunk, options.timeout)
      .then((clean) => {
        if clean:
          this.push(chunk)
        else:
          emit("threat")
          this.destroy()
      })
      .catch(() => {
        # timeout or clamav unreachable
        log("scan timeout, allowing")
        this.push(chunk)
      })
```

## Contract

- **Endpoint**: `POST /uploads` (same as form 01)
- **Content-Type**: `multipart/form-data`
- **File field name**: `file`
- **Request header `Expect: 100-continue`** (optional; 100 Continue 前后端可协商是否接收)
- **Response 201 Created**:
  ```json
  {
    "id": "<storage-key>",
    "url": "<public-or-relative-url>",
    "size": 1234567890,
    "etag": "<etag-or-checksum>"
  }
  ```
- **Response 413 Payload Too Large**: 累计 bytes 超限（流处理中发现）
- **Response 415 Unsupported Media Type**: 魔数嗅探失败
- **Response 451 Unavailable For Legal Reasons**: 病毒扫描检出威胁
- **Response 500 Internal Server Error**: 存储上传失败
- **背压保证**：source 快度受 sink 限制；任一阶段 error 导致 request 暂停，孤儿分片 abort

## Pitfalls

- ❌ **调用 `request.buffer()` / `await req.body()`**
  - **Fix**: 永远保持流。parser emit per-part stream，链接各阶段，不落盘 Buffer。

- ❌ **完整接收后再 sniff**
  - **Fix**: 第一 KB 到达即 sniff 并决策；失败立即 abort source（背压传播）。

- ❌ **病毒扫描同步阻塞热路径**
  - **Fix**: 小文件（< 50 MB）inline 同步；大文件隔离 quarantine bucket + 异步后台扫。

- ❌ **错误时忘记 abort 存储上传**
  - **Fix**: 任何阶段 error → 显式调用 `storage.abortMultipartUpload(uploadId)` 清理孤儿分片。

- ❌ **网关层无 `client_max_body_size` 限制**
  - **Fix**: reverse proxy (nginx) 设 `client_max_body_size 500m`；应用也设 max，二层防护。

- ❌ **无 slow-loris / read timeout**
  - **Fix**: per-connection read timeout (30s default) + minimum rate check (e.g. 1 KB/s minimum)；慢连接被 kick。

- ❌ **背压不传播**（parser 快、sink 慢，内存爆）
  - **Fix**: `.pipe(stage, {end: true})` 链接，不跳过 highWaterMark；监控 parser.paused 状态。

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [node-formidable/formidable](https://github.com/node-formidable/formidable) — ~7k★ (verified 2026-05-28): streaming multipart parser with built-in backpressure and per-part stream emission
- [expressjs/multer](https://github.com/expressjs/multer) — ~12k★ (verified 2026-05-28): stream storage strategy and custom middleware for piped transformations (multer + stream-based sink patterns)
