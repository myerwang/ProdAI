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
├── table/                                     # テーブル設計パターン（8 形式）
│   ├── README.md                              # 索引 + 判断ツリー
│   ├── 01_offset_table/                       # 標準 offset 分頁
│   ├── 02_cursor_pagination/                  # cursor / keyset 分頁
│   ├── 03_infinite_scroll/                    # 無限スクロール
│   ├── 04_virtual_scroll/                     # 仮想スクロール
│   ├── 05_editable_data_grid/                 # Excel-like データグリッド
│   ├── 06_tree_table/                         # ツリーテーブル（ネスト）
│   ├── 07_pivot_table/                        # ピボットテーブル
│   └── 08_server_side_row_model/              # サーバーサイド行モデル（>1M）
├── auth/                                      # 認証パターン（6 形式）
│   ├── README.md                              # 索引 + 判断ツリー
│   ├── 01_session_cookie.md                   # サーバー session + cookie（有状態）
│   ├── 02_jwt.md                              # JWT ステートレストークン
│   ├── 03_oauth2_oidc.md                      # OAuth2 / OIDC 委譲 / 連合ログイン
│   ├── 04_api_key.md                          # API Key / PAT（マシン間）
│   ├── 05_webauthn_passkey.md                 # WebAuthn / Passkeys パスワードレス
│   └── 06_refresh_token_rotation.md           # リフレッシュトークン回転 / セッション寿命
├── form/                                      # フォームパターン（9 形式）
│   ├── README.md                              # 索引 + 判断ツリー + 横断レイヤー
│   ├── 01_single_form.md                      # 単一ページフォーム（controlled/uncontrolled + schema 検証）
│   ├── 02_multistep_wizard.md                 # 多段ウィザード / stepper
│   ├── 03_dynamic_field_array.md              # 動的フィールド配列（行の増減）
│   ├── 04_conditional_survey_form.md          # 条件分岐 / アンケート（skip logic）
│   ├── 05_schema_driven_form.md               # Schema 駆動 / サーバー駆動フォーム
│   ├── 06_inline_edit.md                      # インライン編集（capability）
│   ├── 07_autosave_draft.md                   # 自動保存 / 下書き（capability）
│   ├── 08_search_filter_form.md               # 検索 / フィルターフォーム
│   └── 09_file_upload_form.md                 # ファイルアップロード字段（→ upload/ ポインタ）
└── upload/                                     # ファイルアップロードパターン（8 形式）
    ├── README.md                              # 索引 + 判断ツリー + Summary
    ├── 01_server_proxied_multipart.md         # サーバー代理 multipart（≤5MB / 必ずアプリ層経由）
    ├── 02_presigned_direct_put.md             # 署名付き URL 直送り（5–100MB）
    ├── 03_multipart_parallel_parts.md         # 並列分割 multipart（>100MB / 中止可能）
    ├── 04_resumable_tus.md                    # tus プロトコル再開可能（弱ネット / セッション跨ぎ）
    ├── 05_client_preprocessing.md             # クライアント前処理（resize / HEIC / EXIF / hash）
    ├── 06_background_offline_queue.md         # バックグラウンド離線キュー（SW + IndexedDB / PWA）
    ├── 07_streaming_server_ingestion.md       # サーバー側ストリーミング受信（代理必須でも省メモリ）
    └── 08_post_upload_pipeline.md             # アップロード後派生パイプライン（サムネ / 検疫 / 変換）
```

フォルダは**オンデマンド作成**: 散発的な単発はまず根に置き、3 つ目の関連文書が来た時に
フォルダ化する。事前に空フォルダを作らない。
**例外**: 成熟した多形式タクソノミーと最初から認識できる topic（`table/`、`auth/` 等）は、
最初からフォルダを作り `table/` 同様に人気生産級形式を一気に全収録する（`AGENTS.md` §2.3 参照）。

## 現在の内容

### [`table/`](./table/) — テーブル設計パターン

8 種類のプロダクション級 table 形式: Offset / Cursor / Infinite Scroll /
Virtual Scroll / Editable Data Grid / Tree / Pivot / Server-Side Row Model。
各形式に疑似コード、ライブラリ推奨、pitfalls 含む。

[table/README.md](./table/README.md) の判断ツリーから参照を開始する。

### [`auth/`](./auth/) — 認証パターン

6 種類のプロダクション級認証形式: Session+Cookie / JWT / OAuth2-OIDC / API Key /
WebAuthn-Passkeys / Refresh Token Rotation。各形式に疑似コード、pitfalls、実測高star 参考含む。

[auth/README.md](./auth/README.md) の判断ツリー（まず「有状態 vs ステートレス」）から開始する。

### [`form/`](./form/) — フォームパターン

9 種類のプロダクション級 form 形式: Single / Multi-step Wizard / Dynamic Field Array /
Conditional-Survey / Schema-driven / Inline Edit / Autosave-Draft / Search-Filter /
File Upload。主軸は UX/アーキテクチャ形態、状態管理と検証は各 form 内の「横断レイヤー」
として扱う。各形式に疑似コード、pitfalls、実測高star 参考含む。

[form/README.md](./form/README.md) の判断ツリー（まず「フィールド集合がビルド時に既知か」）から開始する。

### [`upload/`](./upload/) — ファイルアップロードパターン

8 種類のプロダクション級 upload 形式: Server-Proxied Multipart / Presigned Direct PUT /
Multipart Parallel Parts / Resumable tus / Client Preprocessing / Background Offline Queue /
Streaming Server Ingestion / Post-Upload Pipeline。対象は画像 / ファイル / 動画 / バイナリ
blob すべて；フロントエンド / バックエンド / クライアント前処理 / サーバー派生処理の 4 軸
を網羅。`form/09_file_upload_form.md` は「フォーム字段として接続する場合」のポインタに退化、
転送機構は本ディレクトリに収録。各形式に疑似コード、pitfalls、実測高star 参考含む。

[upload/README.md](./upload/README.md) の判断ツリー（まずファイルサイズで分岐、その後 orthogonal layers を重ねる）から開始する。

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
- 2026-05-23 — `auth/` 認証パターン topic を追加（6 形式を全収録、実測高star 参考）; 「成熟した多形式タクソノミーは最初からフォルダ全収録」手順を確立（AGENTS.md §2.3）
- 2026-05-28 — `form/` フォームパターン topic を追加（9 形式を全収録: Single / Wizard / Field Array / Conditional-Survey / Schema-driven / Inline Edit / Autosave / Search-Filter / File Upload; 状態管理と検証は横断レイヤー扱い）; References は 2026-05-28 実測
- 2026-05-28 — `upload/` ファイルアップロード topic を追加（8 形式を全収録: Server-Proxied Multipart / Presigned Direct PUT / Multipart Parallel Parts / Resumable tus / Client Preprocessing / Background Offline Queue / Streaming Server Ingestion / Post-Upload Pipeline）; `form/09_file_upload_form.md` は `upload/` へのポインタに収縮; References は 2026-05-28 実測
