---
topic: upload
applies_to: [frontend, backend, storage]
status: stable
last_reviewed: 2026-05-28
---

# Upload — 生产级文件上传 taxonomy

八个生产级上传 form，覆盖小文件代理、大文件直传、断点续传、客户端预处理、后台离线队列、服务端流式、派生处理全链路。前端、后端、移动端、存储层各有其用，与 `form/` 正交（form/09 仅作"作为表单字段时如何接入"的指针）。

---

## Decision Tree (read first)

```
文件大小？
├─ ≤ 5 MB
│   ├─ 必须经过应用层（业务校验 / 转换 / 审计）  → 01_server_proxied_multipart
│   └─ 直接到存储，无校验                         → 02_presigned_direct_put
├─ 5–100 MB
│   └─ 直传对象存储（presigned URL）             → 02_presigned_direct_put
└─ > 100 MB / 视频
    ├─ 网络可靠，不需跨会话恢复                  → 03_multipart_parallel_parts
    └─ 弱网 / 跨会话 / 长期恢复                  → 04_resumable_tus

Orthogonal layers（可与上面叠加，不互斥）：
├─ 客户端 resize / 转码 / EXIF strip / hash      → 05_client_preprocessing
├─ PWA / tab 关闭也要传 / 后台续传               → 06_background_offline_queue
├─ 必须代理但内存受限                            → 07_streaming_server_ingestion
└─ 上传后派生 / 扫毒 / 转码 / 缩略图             → 08_post_upload_pipeline
```

---

## Summary

| # | Form | When (一句话中文) | applies_to | Primary OSS ≥5k★ |
|---|------|----------------|-----------|-------------------|
| 1 | [server_proxied_multipart](./01_server_proxied_multipart.md) | ≤5MB / 必经应用层校验 | F+B | expressjs/multer ~12k★ |
| 2 | [presigned_direct_put](./02_presigned_direct_put.md) | 5–100MB 直传对象存储 | F+B | aws/aws-sdk-js ~8k★ |
| 3 | [multipart_parallel_parts](./03_multipart_parallel_parts.md) | >100MB 并行分片可中止 | F+B | transloadit/uppy ~31k★ |
| 4 | [resumable_tus](./04_resumable_tus.md) | 弱网 / 跨会话恢复 | F+B | transloadit/uppy ~31k★ |
| 5 | [client_preprocessing](./05_client_preprocessing.md) | 上传前 resize / HEIC / EXIF / hash | F | pqina/filepond ~16k★ |
| 6 | [background_offline_queue](./06_background_offline_queue.md) | PWA / tab 关闭也要传 | F | GoogleChrome/workbox ~13k★ |
| 7 | [streaming_server_ingestion](./07_streaming_server_ingestion.md) | 必须代理但要省内存（流式） | B | node-formidable/formidable ~7k★ |
| 8 | [post_upload_pipeline](./08_post_upload_pipeline.md) | 上传后派生 / 扫毒 / 转码 | B | imgproxy/imgproxy ~11k★ |

---

## Common Combinations

- **用户头像**：05 (客户端 resize) + 02 (presigned PUT) + 08 (缩略图派生)
- **文档批量上传（<100MB）**：01 (server proxied) + 08 (扫毒 / 提取元数据)
- **视频 / 大文件**：03 (multipart parallel) 或 04 (resumable) + 08 (转码派生)
- **离线 PWA**：06 (后台队列) + 01 或 02（网络恢复后补全）
- **流式日志 / 大对象**：07 (streaming server ingestion) + 08 (后处理)

---

## 跨切关注点

- **Authorization scope** — presigned URL 必须 scope 到 key prefix + content-type + size + 短 TTL（5–15 min）
- **Server-side re-validation** — 客户端校验仅 UX；服务端按内容 sniff 类型、size limit、病毒扫描、image-bomb 防护
- **Orphan objects** — 未引用对象 lifecycle 清理；cancel 时显式 DELETE；事务性 finalize
- **Idempotency** — finalize / complete multipart 调用必须带幂等键（dedup client token）
- **CORS headers** — 直传场景 `Access-Control-Expose-Headers: ETag, x-amz-*`；tus 协议完整 header 支持
- **Backpressure / limits** — 并发上限、单用户带宽、总对象数配额、part size 约束

---

## 与 form/ 的关系

`form/09_file_upload_form.md` 是"作为表单字段组件"的视角入口，指向本目录。详细的上传传输机制、存储集成、派生处理流程全部在 `upload/` 各 form 中展开。

---

## References convention

Per `../AGENTS.md` §3.6，每个 form 的 README 必须引用 ≥1 个 GitHub 高星 OSS repo（>5,000★），星数通过 GitHub API 实测，附带 `verified <date>` 戳。无闭门造车、无编造星数。"Primary OSS" 列为快速指针；各 form README 的 `## References` 章节持有验证详情。

---

## History

- **2026-05-28**: Initial taxonomy. 8 forms indexed, decision tree + summary + cross-cutting concerns. 各 form 已单独写入、高星 OSS 参考待审。
