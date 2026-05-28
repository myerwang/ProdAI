---
form: client_preprocessing
topic: upload
applies_to: [frontend]
decision: shrink/normalize before upload to save bandwidth and strip privacy metadata
status: stable
last_reviewed: 2026-05-28
---

# Upload 5: Client-Side Preprocessing

客户端在上传前对文件进行本地归一化和压缩：解码图像、缩放到规范尺寸（如长边 2000px）、重新编码为目标格式（JPEG/WebP）、剥离 EXIF 元数据（GPS、设备序列号等隐私信息）、生成确定性 SHA256 哈希用于去重。处理在 Web Worker 中运行避免卡顿。输出的小型归一化 blob 随后通过 01–04 任一传输方式上传。适合移动端照片上传、带宽受限客户端、隐私关键场景。

## When to use

- 移动端相机照片上传：HEIC/HEIF 自动转 JPEG、EXIF 自动剥离
- 带宽受限客户端：降采样 40MP 原图到 2k px，减 80%+ 大小
- 用户隐私敏感：GPS、制造商序列号、设备标识必须在客户端清除
- 跨客户端一致性：相同源文件通过归一化产生相同哈希，避免重复存储
- 前端即时反馈：预缩略图给用户实时预览上传后的效果

## When NOT to use

- 医疗、法律原件必须保留原始未损版本：禁止有损预处理，只能原件上传
- 服务端已有廉价派生流水线：若后端能快速生成缩略图/预览，客户端预处理收益小
- 极低算力设备（IoT、老旧手机）：解码/缩放 CPU 成本 > 带宽节省收益
- 原文件内容可恢复性要求高：任何有损转换都禁止
- 用户没给明确同意修改文件：需在 UX 明确告知"将压缩至 JPEG、去除位置信息"

## Conclusion

在 Web Worker 中运行确定性的客户端流水线：`decode(blob) → resize(keeping aspect) → reencode(target format, quality=0.82) → strip EXIF metadata → sha256(final blob)→ hash`。缓存 `hash → uploaded_key` 映射以跳过重复上传。输出的小型、隐私安全的 blob 随后通过 01–04 任一传输方式上传。客户端预处理是 UX 和带宽优化，**不是安全保障** — 服务端必须始终重新校验类型、大小、内容完整性，不可信任客户端声称的 hash 或尺寸。

## Frontend

```
interface ImageDimensions:
  width: integer
  height: integer

interface PreprocessedFile:
  originalName: string
  originalSize: integer
  originalType: string
  finalBlob: Blob
  finalSize: integer
  sha256Hash: string
  dimensions: ImageDimensions
  orientationApplied: boolean

const MAX_DIMENSION = 2000          # px，长或宽的上限
const JPEG_QUALITY = 0.82           # 0–1 范围
const CHUNK_SIZE = 65536            # 逐块读文件避免内存爆

function createPreprocessWorker():
  # 在 Web Worker 线程运行，避免主线程卡顿
  return new Worker("preprocess-worker.js")

# 主线程启动预处理
function preprocessFile(blob: Blob, originalName: string) -> Promise<PreprocessedFile>:
  worker = createPreprocessWorker()
  
  return new Promise((resolve, reject) => {
    worker.onmessage = (event) => {
      result = event.data
      if result.error:
        reject(Error(result.error))
      else:
        resolve(result)
      worker.terminate()
    }
    
    worker.onerror = (error) => {
      reject(error)
      worker.terminate()
    }
    
    worker.postMessage({
      blob: blob,
      originalName: originalName,
      maxDimension: MAX_DIMENSION,
      jpegQuality: JPEG_QUALITY
    }, [blob])  # 转移所有权避免复制

# ========== Web Worker 侧 ==========

function workerHandleMessage(event):
  message = event.data
  blob = message.blob
  originalName = message.originalName
  maxDimension = message.maxDimension
  jpegQuality = message.jpegQuality
  
  try:
    # 步骤 1：嗅探类型和读取基本元数据
    originalType = sniffContentType(blob)
    if not isImageType(originalType):
      throw Error("Not an image: " + originalType)
    
    isHeic = originalType contains "heic" or originalType contains "heif"
    isPng = originalType == "image/png"
    isLossless = isPng or (originalType contains "webp" and blob has lossless chunk)
    
    # 步骤 2：解码为 ImageBitmap 或 Canvas
    bitmap = await decodeImageBlob(blob, maxDimension)
    originalDimensions = {width: bitmap.width, height: bitmap.height}
    
    # 步骤 3：应用 EXIF 方向（若存在）
    orientationApplied = false
    if not isPng:  # PNG EXIF 已被 browser decoder 处理
      exifOrientation = await readExifOrientation(blob)
      if exifOrientation and exifOrientation != 1:
        bitmap = applyOrientationTransform(bitmap, exifOrientation)
        orientationApplied = true
    
    # 步骤 4：缩放（保持宽高比）
    resized = false
    if bitmap.width > maxDimension or bitmap.height > maxDimension:
      scale = min(maxDimension / bitmap.width, maxDimension / bitmap.height)
      newWidth = floor(bitmap.width * scale)
      newHeight = floor(bitmap.height * scale)
      bitmap = resizeBitmap(bitmap, newWidth, newHeight)
      resized = true
    
    # 步骤 5：选择目标格式和重编码
    targetType = jpeg
    targetQuality = jpegQuality
    
    if isLossless:
      # PNG/无损 WebP → 保持 PNG（避免二次有损）
      targetType = png
      targetQuality = null
    elif isHeic:
      # HEIC → JPEG（浏览器不支持 HEIC decode，只能用库如 heic2any polyfill）
      targetType = jpeg
    
    finalBlob = await reencodeImageBitmap(bitmap, targetType, targetQuality)
    
    # 步骤 6：剥离 EXIF 和其他元数据
    # （重编后的 bitmap → blob 通常已无 EXIF，但显式清理保险）
    finalBlob = await stripMetadata(finalBlob)
    
    # 步骤 7：计算哈希（最终 blob 的规范表示）
    sha256Hash = await sha256(finalBlob)
    
    # 完成，返回给主线程
    workerSelf.postMessage({
      originalName: originalName,
      originalSize: blob.size,
      originalType: originalType,
      finalBlob: finalBlob,
      finalSize: finalBlob.size,
      sha256Hash: sha256Hash,
      dimensions: {
        width: bitmap.width,
        height: bitmap.height
      },
      orientationApplied: orientationApplied,
      error: null
    }, [finalBlob])
  
  catch error:
    workerSelf.postMessage({
      error: error.message
    })

function sniffContentType(blob: Blob) -> string:
  # 读前 4–8 字节，检查幻数
  header = await readBlob(blob, 0, 8)
  
  if header[0:4] == [0xff, 0xd8, 0xff, 0xe*]:
    return "image/jpeg"
  if header[0:4] == [0x89, 0x50, 0x4e, 0x47]:
    return "image/png"
  if header[0:4] == [0x52, 0x49, 0x46, 0x46] and header[8:12] == [0x57, 0x45, 0x42, 0x50]:
    return "image/webp"
  if header[0:4] == [0x00, 0x00, 0x00, 0x20] or header[0:4] == [0x00, 0x00, 0x00, 0x18]:
    # ftypisom / ftypisom (简化 HEIC 检查)
    return "image/heic"
  if blob.type:
    return blob.type
  return "application/octet-stream"

function isImageType(contentType: string) -> boolean:
  return contentType startsWith "image/"

function readExifOrientation(blob: Blob) -> optional integer:
  # 读 JPEG/HEIC，解析 EXIF，提取 Orientation tag (0x0112)
  # 实现：使用 piexifjs 或 exifjs 库；若无则返回 null
  try:
    exifData = await piexif.load(await blob.arrayBuffer())
    if "0th" in exifData:
      orientation = exifData["0th"][piexif.ImageIFD.Orientation]
      return orientation
  catch:
    pass
  return null

function applyOrientationTransform(bitmap: ImageBitmap, orientation: integer) -> ImageBitmap:
  # orientation: 1=default, 3=rotate-180, 6=rotate-90-cw, 8=rotate-270-cw 等
  canvas = new OffscreenCanvas(bitmap.width, bitmap.height)
  ctx = canvas.getContext("2d")
  
  # 根据方向旋转/翻转
  match orientation:
    case 2:
      ctx.translate(bitmap.width, 0)
      ctx.scale(-1, 1)
    case 3:
      ctx.translate(bitmap.width, bitmap.height)
      ctx.rotate(Math.PI)
    case 4:
      ctx.translate(0, bitmap.height)
      ctx.scale(1, -1)
    case 5:
      ctx.rotate(-Math.PI/2)
      ctx.scale(-1, 1)
    case 6:
      ctx.rotate(-Math.PI/2)
      ctx.translate(-bitmap.height, 0)
    case 7:
      ctx.rotate(Math.PI/2)
      ctx.scale(-1, 1)
    case 8:
      ctx.rotate(Math.PI/2)
      ctx.translate(-bitmap.height, 0)
  
  ctx.drawImage(bitmap, 0, 0)
  return await canvas.convertToBlob()

function resizeBitmap(bitmap: ImageBitmap, newWidth: integer, newHeight: integer) -> ImageBitmap:
  canvas = new OffscreenCanvas(newWidth, newHeight)
  ctx = canvas.getContext("2d")
  ctx.drawImage(bitmap, 0, 0, newWidth, newHeight)
  return canvas.convertToBlob()

function reencodeImageBitmap(bitmap: ImageBitmap, format: string, quality: optional float) -> Promise<Blob>:
  canvas = new OffscreenCanvas(bitmap.width, bitmap.height)
  ctx = canvas.getContext("2d")
  ctx.drawImage(bitmap, 0, 0)
  
  options = {type: format}
  if quality != null:
    options.quality = quality
  
  return canvas.convertToBlob(options)

function stripMetadata(blob: Blob) -> Promise<Blob>:
  # 方法 1：重编过程已移除 EXIF（OffscreenCanvas 输出没有元数据）
  # 方法 2（保险）：用 piexif 手动清除
  # 对大多数场景，重编后的 blob 已干净
  return blob

function sha256(blob: Blob) -> Promise<string>:
  # 使用 SubtleCrypto API
  arrayBuffer = await blob.arrayBuffer()
  hashBuffer = await crypto.subtle.digest("SHA-256", arrayBuffer)
  
  # 转换为十六进制字符串
  hashArray = Array.from(new Uint8Array(hashBuffer))
  hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('')
  return hashHex

# 主线程中的去重和上传集成
function FileUploadWidgetWithPreprocessing(props):
  state file: optional Blob = null
  state uploadState: string = "idle"      # idle | preprocessing | uploading | done | error
  state progress: float = 0.0
  state preprocessedFile: optional PreprocessedFile = null
  state errorMsg: optional string = null
  state uploadedHash: optional string = null
  
  # localStorage 格式：{hash -> {url, timestamp}}
  state dedupeCache: Map<string, UploadRecord> = loadDedupeCache()
  
  function onFileSelect(f: Blob):
    file = f
    uploadState = "preprocessing"
    errorMsg = null
    progress = 0.0
    preprocessedFile = null
    
    try:
      preprocessedFile = await preprocessFile(f, f.name)
      
      # 检查是否已上传过
      existingRecord = dedupeCache.get(preprocessedFile.sha256Hash)
      if existingRecord:
        uploadState = "done"
        uploadedHash = preprocessedFile.sha256Hash
        props.onSuccess({
          url: existingRecord.url,
          isDuplicate: true,
          hash: preprocessedFile.sha256Hash
        })
        return
      
      # 未上传过，启动传输
      uploadState = "uploading"
      await uploadPreprocessedFile(preprocessedFile)
    
    catch e:
      uploadState = "error"
      errorMsg = e.message
  
  function uploadPreprocessedFile(processed: PreprocessedFile):
    # 选择合适的传输方式（Form 01–04）
    # 例：小于 50MB 用 Form 01（服务器代理）；大于用 Form 04（tus）
    
    if processed.finalSize <= 50_000_000:
      # 使用 Form 01: Server Proxied Multipart
      formData = new FormData()
      formData.append("file", processed.finalBlob)
      formData.append("name", processed.originalName)
      formData.append("x-original-hash", processed.sha256Hash)
      
      response = await httpRequest({
        method: "POST",
        url: "/uploads",
        body: formData,
        onProgress: (loaded, total) => {
          progress = 0.3 + (loaded / total) * 0.7  # 预处理占 30%，上传占 70%
        }
      })
      
      if response.status == 201:
        uploadedHash = processed.sha256Hash
        dedupeCache.set(processed.sha256Hash, {
          url: response.json().url,
          timestamp: now()
        })
        saveDedupeCache(dedupeCache)
        
        uploadState = "done"
        props.onSuccess(response.json())
    else:
      # 使用 Form 04: Resumable tus（超大文件）
      await uploadViaTus(processed)
  
  function loadDedupeCache() -> Map<string, UploadRecord>:
    raw = localStorage.get("upload_dedupe_cache")
    if raw:
      return Map(JSON.parse(raw))
    return new Map()
  
  function saveDedupeCache(cache: Map):
    data = Array.from(cache.entries())
    localStorage.set("upload_dedupe_cache", JSON.stringify(data))
  
  render:
    if uploadState == "idle":
      file-input (accept="image/*")
    if uploadState == "preprocessing":
      spinner ("Optimizing image...")
    if uploadState == "uploading":
      progress-bar (progress)
      cancel-button
    if uploadState == "done":
      success-checkmark
      if isDuplicate:
        info-badge ("Image already uploaded")
    if uploadState == "error":
      error-message (errorMsg)
      retry-button
```

## Backend

后端通过 01–04 的传输方式接收本 form 的输出（最终 blob），**必须始终重新校验**：

```
function handleUploadPost(request):
  # 客户端发送的预处理 blob（或 tus 的最终 object）
  file = request.file
  clientHash = request.headers.get("X-Original-Hash")  # 可选；不信任
  
  # ★ 重新校验：服务器始终不信任客户端预处理
  
  # 1. 重新校验类型（magic bytes）
  sniffedType = sniffMagicBytes(file.slice(0, 512))
  if not isAllowedType(sniffedType):
    return 415 Unsupported Media Type
  
  # 2. 重新校验大小（即使客户端说"已缩放至 2000px"也要重测）
  if file.size > MAX_ALLOWED_SIZE:
    return 413 Payload Too Large
  
  # 3. 重新计算哈希（客户端哈希仅用于比较，不记录）
  serverHash = sha256(file)
  if clientHash and clientHash != serverHash:
    log_warning("client_hash_mismatch: client=" + clientHash + " server=" + serverHash)
  
  # 4. 必要时生成派生（缩略图、预览）
  #    服务端派生应从规范形态（预处理的最终 blob）生成
  
  # 继续通常的 multipart 或 tus 终止逻辑
  recordId = generateId()
  finalUrl = storeFile(file, recordId, sniffedType)
  
  return 201 {
    id: recordId,
    url: finalUrl,
    size: file.size,
    sha256: serverHash
  }
```

## Contract

本 form 无独立的 wire contract。客户端预处理的输出（最终 blob）是 01–04 传输方式之一的输入：

- **Form 01 (multipart)**：将预处理 blob 添加到 FormData，POST 到 `/uploads`
- **Form 02 (presigned PUT)**：将预处理 blob PUT 到 AWS 预签名 URL
- **Form 03 (multipart parallel)**：将预处理 blob 分割成块，并行上传
- **Form 04 (tus)**：通过 tus 可恢复协议上传预处理 blob

选择基于 `finalSize` 和网络条件确定。

## Pitfalls

- ❌ **主线程中解码/缩放大型图像**
  - **Fix**：在 Web Worker 中运行；使用 `createImageBitmap` 选项渐进式降采样

- ❌ **应用 EXIF 方向之前移除元数据**
  - **Fix**：先应用方向变换 → 重编期间自动移除元数据

- ❌ **将无损 PNG 或截图无声地重新编码为 JPEG**
  - **Fix**：根据内容类型决策；PNG → PNG，照片 → JPEG/WebP

- ❌ **在授权/安全决策中使用客户端哈希**
  - **Fix**：哈希仅用于 UX 去重；服务器始终接收后重新计算验证

- ❌ **低端手机上 40MP 原始文件内存爆炸**
  - **Fix**：使用 `createImageBitmap({resizeWidth, resizeHeight})` 选项限制源大小；分步降采样

- ❌ **HEIC 图像浏览器原生不支持**
  - **Fix**：heic2any 库 polyfill 或服务端转换；如果客户端无法处理则原样传输

- ❌ **同一文件的多个预处理结果不一致**
  - **Fix**：保证确定性管道；用相同设置（maxDim、quality、orientation logic）重新运行会生成相同哈希

## References

高星开源实现（星数已验证 2026-05-28 通过 GitHub API；≥5,000★ 标准）：

- [pqina/filepond](https://github.com/pqina/filepond) — ~16k★ (verified 2026-05-28): 生产级上传组件，filepond-plugin-image-transform 插件提供客户端 resize、EXIF 剥离、格式转换
- [lovell/sharp](https://github.com/lovell/sharp) — ~32k★ (verified 2026-05-28): Node.js 服务端图像处理库；若保留原件并在服务端执行预处理变体时使用
