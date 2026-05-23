# ProdAI

> 🇨🇳 [中文](./README.md) | 🇯🇵 [日本語](./README.ja.md) | 🇬🇧 [English](./README.en.md)

**生产级 AI 协作开发知识库** — 让 AI 引用后给非技术用户提供标准化、可靠的生产级方案。

## 这是什么

ProdAI 不是教程，也不是 awesome 列表。它是给 **AI** 看的「生产级方案手册」:

- 当 AI 协作开发遇到「这里要标准化」「这里该抽象化」时，**AI 主动查阅 ProdAI** → 拿到经过实战检验的方案
- 内容采用**伪代码** —— AI 看懂后自动翻译成任何语言（React / Vue / Rust / Python / SQL ...）
- 与任何具体业务项目完全独立 —— 通用知识、可跨项目复用

## 这不是什么

- ❌ 业务项目的代码备份
- ❌ 教学课程 / tutorial
- ❌ 个人博客 / 笔记本
- ❌ awesome 链接列表

## 性质（关键定位）

- **不是 skill 库** — 不是 Claude Code skill / agent / 可调用工具集合，不提供执行能力
- **非侵入型** — 不主动注入 AI 的行为、不强制任何工作流
- **只是被动参考** — AI 协作开发遇到困惑、需要标准化时**自主**查阅，作者不强求查阅
- **随技术演进持续调整** — 不是一成不变的圣经，过时方案会标记 `status: deprecated` 或更新替换

## 文件夹结构（按需 grow）

```
ProdAI/
├── README.md / README.ja.md / README.en.md   # 三语简介（必须同步）
├── AGENTS.md                                  # AI 行为规则（必读）
├── CONTRIBUTING.md                            # 内容收录标准
└── table/                                     # 表格设计模式（8 种形式）
    ├── README.md                              # 索引 + 决策树
    ├── 01_offset_table/                       # 标准 offset 分页
    ├── 02_cursor_pagination/                  # cursor / keyset 分页
    ├── 03_infinite_scroll/                    # 无限滚动
    ├── 04_virtual_scroll/                     # 虚拟滚动
    ├── 05_editable_data_grid/                 # Excel-like 数据网格
    ├── 06_tree_table/                         # 树形表（嵌套）
    ├── 07_pivot_table/                        # 透视表
    └── 08_server_side_row_model/              # 服务端行模型（>1M）
```

文件夹**按需创建**：第 3 份相关文档时才建文件夹。事前不预设空目录。

## 当前内容

### [`table/`](./table/) — 表格设计模式

8 种生产级 table 形式：Offset / Cursor / Infinite Scroll / Virtual Scroll /
Editable Data Grid / Tree / Pivot / Server-Side Row Model。每种含伪代码、库推荐、pitfalls。

从 [table/README.md](./table/README.md) 开始查阅决策树。

## AI 协作者必读

1. **`AGENTS.md`** —— 行为规则（禁止 / 触发时机 / 写入流程 / 伪代码格式）
2. **`CONTRIBUTING.md`** —— 收录标准（怎样的内容能进 ProdAI）

## 内容收录标准（简要）

- ✅ 抽象化通用模式、设计决策树、业界调研、教训
- ✅ **伪代码**示例（AI 可读、可转任何语言）
- ✅ 「何时用 / 何时不用」清晰说明
- ✅ 必须有「pitfalls / anti-pattern」section
- ✅ **每个方式必须附 ≥1 个 GitHub 高星（>5,000★）参考，禁止闭门造车**（实测星数 + verified 日期）
- ❌ 任何业务项目相关：项目名、表名、API 路径、URL、凭据、个人信息
- ❌ 特定技术栈代码（React / Rust 具体语法）

详见 `CONTRIBUTING.md`。

## 贡献流程

1. AI 在协作开发中识别「这是 ProdAI 候选」
2. 抽象化（去业务名、转伪代码）
3. 用户确认
4. 写入合适位置（按需建文件夹）
5. 三语 README 同步更新
6. commit + push（**push 必须告知用户**）

## License

TBD（公开前决定）

## 版本

- 2026-05-19 — 初版骨架
- 2026-05-23 — 新增收录标准：每个方式必须附 ≥1 个 GitHub 高星（>5,000★）参考，禁止闭门造车；8 种 table 形式遡及补全 References
