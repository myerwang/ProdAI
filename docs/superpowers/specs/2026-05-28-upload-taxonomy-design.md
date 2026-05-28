---
spec: upload-taxonomy
topic: upload
applies_to: [frontend, backend, storage]
status: approved
created: 2026-05-28
author: brainstorming session
---

# Spec — `upload/` 多形式 taxonomy（前后端文件上传）

## 1. 目的

在 ProdAI 现有 `table/` `auth/` `form/` 之外，新增 `upload/` 顶层 topic 文件夹，
按 AGENTS.md §2.3 "認知済み多形式タクソノミー" 模式一次性收录 8 个生产级上传形式。
覆盖图片、文件、视频、二进制 blob 任意载体，故 topic 名采用通用的 `upload/`
而非 `file-upload/`。

## 2. 与现有内容的关系

- 现有 `form/09_file_upload_form.md` 是从"表单字段"视角对上传的概述。
  本 taxonomy 上线后，`form/09_file_upload_form.md` **保留**，但内容收缩为
  1 段总览 + 一句"详见 [../upload/README.md](../upload/README.md)"。
- 不在 `form/` 内拆分，因为上传机制本身与"表单"正交：上传可以独立存在
  （消息附件、富文本图片、上传中心）。`form/` 只承担"作为表单字段时如何接入"。

## 3. 收录范围（8 个 form，§A）

```
upload/
├── README.md
├── 01_server_proxied_multipart.md
├── 02_presigned_direct_put.md
├── 03_multipart_parallel_parts.md
├── 04_resumable_tus.md
├── 05_client_preprocessing.md
├── 06_background_offline_queue.md
├── 07_streaming_server_ingestion.md
└── 08_post_upload_pipeline.md
```

每个 NN 文件均符合 AGENTS.md §3 全部要求：metadata / 结论先出し / 伪代码 /
Pitfalls / 5,000★+ References。

## 4. 各 Form metadata 与主参考（§B）

GitHub star 实测于 2026-05-28（详见调研记录）。

| # | Form | applies_to | 定位 | Primary OSS ≥5k★ |
|---|------|------------|------|-------------------|
| 1 | server_proxied_multipart | [frontend, backend] | 默认起点：小文件 / 强业务校验 / 必经应用层 | expressjs/multer ~12k★ |
| 2 | presigned_direct_put | [frontend, backend] | 中等文件 / 卸载带宽到对象存储 | aws/aws-sdk-js ~8k★; transloadit/uppy ~31k★ (AwsS3) |
| 3 | multipart_parallel_parts | [frontend, backend] | 大文件 (>100MB) / 并行分片 / 可中止 | transloadit/uppy ~31k★ (AwsS3Multipart) |
| 4 | resumable_tus | [frontend, backend] | 弱网/移动/跨会话恢复 / 大视频 | transloadit/uppy ~31k★ (Tus plugin) |
| 5 | client_preprocessing | [frontend] | 上传前 resize / HEIC→JPEG / EXIF strip / hash 去重 | pqina/filepond ~16k★ (image-transform); lovell/sharp ~32k★（服务端中继预处理变体） |
| 6 | background_offline_queue | [frontend] | PWA / tab 关闭也要继续 / 弱网持久化 | GoogleChrome/workbox ~13k★; transloadit/uppy ~31k★ (Golden Retriever) |
| 7 | streaming_server_ingestion | [backend] | 必须代理但要避免内存爆 / 边接边转发 | node-formidable/formidable ~7k★; expressjs/multer ~12k★ |
| 8 | post_upload_pipeline | [backend] | 派生处理：缩略图/扫毒/转码/CDN 预热 | imgproxy/imgproxy ~11k★; h2non/imaginary ~6k★; lovell/sharp ~32k★ |

**说明**：tus 协议的 canonical implementation (tus/tus-js-client 2.5k, tus/tus-node-server 1k)
本身低于 5k★，但 transloadit/uppy (~31k★) 内置 Tus plugin 是 tus 协议在前端的最高 star 落地。
References 仅列 ≥5k★ 项，protocol 名称在正文中提及但不作为孤立链接。S3 multipart 同理。

## 5. NN 文件内部结构

每个 NN_*.md 统一骨架：

```
---
form: <slug>
topic: upload
applies_to: [frontend, backend]   # 或单边（见 §4）
decision: <一句话决策定位>
status: stable
last_reviewed: 2026-05-28
---

# Upload N: <Title>

## When to use
## When NOT to use
## Conclusion

## Frontend (伪代码)
<选择/进度/重试/取消/状态机>

## Backend (伪代码)
<签名/校验/落存储/finalize/事件出口>

## Contract（前后端约定）
<URL / method / headers / 响应体 / 错误码 / CORS / 授权范围>

## Pitfalls
<§E 跨切关注点的本 form 适配 + 本 form 特有陷阱>

## References
<≥1 件 5k★+ OSS, verified 2026-05-28>
```

## 6. 决策树（写进 `upload/README.md`，§C）

```
Q1: 文件大小？
  ≤ 5 MB        → Q2
  5–100 MB      → 02 presigned_direct_put
  > 100 MB / 视频 → Q3

Q2: 必须经过应用层（业务校验/转换/审计）？
  Yes → 01 server_proxied_multipart（+ 07 streaming 防 OOM）
  No  → 02 presigned_direct_put

Q3: 网络可靠 & 不需要跨会话恢复？
  Yes → 03 multipart_parallel_parts
  No  → 04 resumable_tus

Orthogonal layers（可与上面叠加，不互斥）：
  - 客户端 resize/转码/隐私 strip → 加 05
  - PWA / 离线 / 后台续传        → 加 06
  - 上传后派生缩略图/转码/扫毒    → 加 08
```

## 7. `upload/README.md` 骨架（§D）

1. metadata
2. 一句话定位
3. 决策树（§6）
4. Summary 表：列 = Form / When / Primary OSS / 决策轴位
5. 跨切关注点清单（§8）

## 8. 跨切关注点（每个 NN Pitfalls 必须点到，§E）

- **Authorization scope**：presigned/upload URL 必须 scope 到 key prefix + content-type + size + 短 TTL
- **Server-side re-validation**：客户端校验仅 UX；服务端必须按内容 sniff 类型 / size / 病毒扫描 / image-bomb 防御
- **Orphan objects**：未引用对象的 TTL / lifecycle 清理；用户取消时显式 DELETE
- **Idempotency**：finalize 调用必须带幂等键，避免重复提交多算 1 份
- **CORS**：直传场景的 `Access-Control-Expose-Headers: ETag` 与 preflight 配置
- **Backpressure / limits**：并发上限、单用户带宽配额、总对象数配额

## 9. 三语 README 同步

新增 `upload/` 后必须同时更新：
- `README.md`（中文，主入口）
- `README.ja.md`
- `README.en.md`

三处的目录列表 + topic 简介保持一致。

## 10. 实现交付物清单

- [ ] `upload/README.md`
- [ ] `upload/01_server_proxied_multipart.md`
- [ ] `upload/02_presigned_direct_put.md`
- [ ] `upload/03_multipart_parallel_parts.md`
- [ ] `upload/04_resumable_tus.md`
- [ ] `upload/05_client_preprocessing.md`
- [ ] `upload/06_background_offline_queue.md`
- [ ] `upload/07_streaming_server_ingestion.md`
- [ ] `upload/08_post_upload_pipeline.md`
- [ ] `form/09_file_upload_form.md` 收缩为 1 段总览 + 指向 `../upload/README.md`
- [ ] `README.md` / `README.ja.md` / `README.en.md` 三语同步更新
- [ ] 一次 commit（AI 自动可），push 前一行告知用户（AGENTS.md §7）

## 11. 非目标（Out of Scope）

- 不在本 spec 内提供任何特定语言（Node/Go/Python/Rust）的可运行代码——按 AGENTS.md §5 全部伪代码
- 不深入 S3 / GCS / R2 / Azure Blob 的具体 API 差异，仅以"对象存储"抽象呈现
- 不收录 BitTorrent / WebRTC DataChannel / WebDAV 等小众传输协议
- 不写运维侧的 Terraform / CDK 部署样例
