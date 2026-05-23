# ProdAI

> 🇨🇳 [中文](./README.md) | 🇯🇵 [日本語](./README.ja.md) | 🇬🇧 [English](./README.en.md)

**プロダクション級 AI 協働開発ナレッジ庫** — AI が参照することで非技術ユーザーにも
標準化された信頼性の高いプロダクション級ソリューションを提供する。

## これは何か

ProdAI はチュートリアルでも awesome リストでもない。**AI** が読むための
「プロダクション級方案ハンドブック」:

- AI 協働開発で「ここは標準化が必要」「ここは抽象化できる」というタイミングで、
  **AI が能動的に ProdAI を参照** → 実戦検証済みの方案を得る
- 内容は **疑似コード** —— AI が読んで任意の言語に翻訳できる（React / Vue / Rust / Python / SQL ...）
- 具体的な業務プロジェクトから完全独立 —— 汎用ナレッジ、プロジェクト横断で再利用可能

## これは何ではないか

- ❌ 業務プロジェクトのコードバックアップ
- ❌ 教学コース / チュートリアル
- ❌ 個人ブログ / 個人ノート
- ❌ awesome リンク集

## 性質（重要な位置付け）

- **skill ライブラリではない** — Claude Code skill / agent / 呼び出し可能ツール集合ではない、実行機能を提供しない
- **非侵入型** — AI の行動に能動的に注入しない、いかなるワークフローも強制しない
- **受動的な参照資料のみ** — AI が協働開発で迷い、標準化が必要な時に**自主的に**参照する。作者は参照を強要しない
- **技術進化と共に継続調整** — 不変の聖典ではない。陳腐化した方案は `status: deprecated` マーク或いは更新置換

## フォルダ構造（オンデマンドで grow）

```
ProdAI/
├── README.md / README.ja.md / README.en.md   # 三語簡介（常に同期）
├── AGENTS.md                                  # AI 行動規則（必読）
├── CONTRIBUTING.md                            # 内容収録基準
└── table/                                     # テーブル設計パターン（8 形式）
    ├── README.md                              # 索引 + 判断ツリー
    ├── 01_offset_table/                       # 標準 offset 分頁
    ├── 02_cursor_pagination/                  # cursor / keyset 分頁
    ├── 03_infinite_scroll/                    # 無限スクロール
    ├── 04_virtual_scroll/                     # 仮想スクロール
    ├── 05_editable_data_grid/                 # Excel-like データグリッド
    ├── 06_tree_table/                         # ツリーテーブル（ネスト）
    ├── 07_pivot_table/                        # ピボットテーブル
    └── 08_server_side_row_model/              # サーバーサイド行モデル（>1M）
```

フォルダは**オンデマンド作成**: 3 つ目の関連文書が来た時に初めてフォルダ化する。
事前に空フォルダを作らない。

## 現在の内容

### [`table/`](./table/) — テーブル設計パターン

8 種類のプロダクション級 table 形式: Offset / Cursor / Infinite Scroll /
Virtual Scroll / Editable Data Grid / Tree / Pivot / Server-Side Row Model。
各形式に疑似コード、ライブラリ推奨、pitfalls 含む。

[table/README.md](./table/README.md) の判断ツリーから参照を開始する。

## AI 協働者必読

1. **`AGENTS.md`** — 行動規則（禁止 / 触発タイミング / 書き込みフロー / 疑似コード形式）
2. **`CONTRIBUTING.md`** — 収録基準（どのような内容が ProdAI に入るか）

## 内容収録基準（簡要）

- ✅ 抽象化された汎用パターン、設計判断ツリー、業界調査、教訓
- ✅ **疑似コード**サンプル（AI が読めて任意言語に変換可能）
- ✅ 「いつ使う / いつ使わない」明確な説明
- ✅ 必ず「pitfalls / アンチパターン」セクションを含む
- ✅ **各方式に GitHub 高star（>5,000★）参考を ≥1 件必須、闭门造车禁止**（star 実測 + verified 日付）
- ❌ 業務プロジェクト関連: プロジェクト名、テーブル名、API パス、URL、凭据、個人情報
- ❌ 特定技術スタックのコード（React / Rust 具体構文）

詳細は `CONTRIBUTING.md` 参照。

## 寄稿フロー

1. AI が協働開発中に「これは ProdAI 候補」と認識
2. 抽象化（業務名除去、疑似コード化）
3. ユーザー確認
4. 適切な場所へ書き込み（必要に応じてフォルダ作成）
5. 三語 README を同期更新
6. commit + push（**push は必ずユーザーに告知**）

## License

TBD（公開前に決定）

## バージョン

- 2026-05-19 — 初版骨格
- 2026-05-23 — 収録基準追加: 各方式に GitHub 高star（>5,000★）参考 ≥1 件必須、闭门造车禁止; 8 種 table 形式に References を遡及付与
