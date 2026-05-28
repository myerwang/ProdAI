---
form: file_upload_form
topic: form
applies_to: [frontend, backend]
field_set: file(s) / binary
decision: 收集 file(s)/binary 作为表单字段；传输机制详见 upload/
status: stable
last_reviewed: 2026-05-28
---

# Form 9: File Upload Form

值为一个或多个 file 的表单字段。从**表单**视角看 contract 很简单：先上传字节、握住引用 (id/handle)、提交 references —— 而不是 bytes。任何上传中的 item 存在时，禁止提交表单。

实际的传输机制（presigned direct PUT vs multipart parallel parts vs resumable tus vs server-proxied multipart，加上客户端预处理、离线队列、流式后端、上传后派生）是独立的成熟 taxonomy，统一收录在 **[../upload/README.md](../upload/README.md)**。

## Form-side contract

- 文件选中即开始上传；字段值 = 已完成 handle 的列表
- 客户端 UX 校验 type/size；服务端必须重新校验（详见 `upload/`）
- 任何上传 item 状态为 `uploading` 时阻断 form submit
- 提交时只发 handles，绝不发 bytes
- 取消 / 删除时同步删除已上传的 orphan object（或依赖 lifecycle TTL）

## 与 upload/ 的分工

| 关注点 | 归属 |
|--------|------|
| 字段值形态、UI 校验、与其他表单字段的协作、提交时机 | 本 form |
| 传输协议、签名、分片、断点、客户端预处理、后端 ingestion、派生处理 | [../upload/](../upload/README.md) |

## References

具体上传机制的 OSS 实现详见 [../upload/README.md](../upload/README.md)。Form-side 上传组件:

- [transloadit/uppy](https://github.com/transloadit/uppy) — ~31k★ (verified 2026-05-28): 带 dashboard UI 的 file uploader，覆盖全部传输机制
- [pqina/filepond](https://github.com/pqina/filepond) — ~16k★ (verified 2026-05-28): 即插即用的 upload 字段，支持 chunking 与 preview
