# ProdAI — AI 行動規則 (AGENTS.md)

ProdAI は **AI が参照して生産級ソリューションを提供するための汎用ナレッジ庫**。
業務プロジェクトから完全独立。受信者は AI（AI を介して非技術ユーザーが標準機能を作る）。

本ファイル (`AGENTS.md`) は ProdAI 内で作業する全ての AI が **無条件遵守** すべき規則。
複数 AI tool (Cursor / Aider / Continue / etc) が AGENTS.md を共通の規則ファイル名として
emerge させているため、tool 非依存の通用名を採用。

ローカル開発者が Claude Code を使う場合は `.claude/CLAUDE.md` に AGENTS.md への参照
（または同一内容のコピー）を置いてもよい。ただし `.claude/` 自体は `.gitignore` で除外
し、リポジトリには **AGENTS.md のみ** をコミットする。

---

## 1. 絶対禁止（即時拒否、議論なし）

業務プロジェクトのいかなる内容も ProdAI に持ち込まない。

1. **業務識別子** — プロジェクト名 / DB テーブル名 / API パス / enum 値 / 内部コンポーネント名
2. **業務領域語** — 業界専門語 / 業務フロー名 / 固有名詞 / 商標 / 提携先
3. **インフラ識別子** — クラウドプロジェクト ID / サービス名 / URL / ホスト / Tenant ID
4. **凭据** — DB 接続情報 / トークン / シークレット（dev 環境含めいかなる形でも）
5. **個人・組織情報** — メール / 個人氏名 / 内部対話の引用 / 顧客情報
6. **業務履歴** — dev_stats / progress / 業務 commit message のコピー
7. **業務コード直コピー** — 業務名込みのコードは絶対 NG（伪代码化は §5 参照）

違反検知時: 即時 user に報告 + 該当ファイル削除。push 済なら git filter-repo / BFG で履歴除去。

---

## 2. 構造ルール

### 2.1 根目录三语 README（必須・常に同期）

```
ProdAI/
├── README.md        ← 中文版（主入口）
├── README.ja.md     ← 日本語版
├── README.en.md     ← English 版
```

3 ファイル**内容完全同期**。一方を更新したら他 2 つも同時更新。

### 2.2 文件夹按需 grow（**predefined しない**）

初期は最小限。3 つ目の関連文書を作る時に初めて文件夹を切る。

```
初期:
ProdAI/
├── README.* (3)
├── .claude/CLAUDE.md
├── CONTRIBUTING.md
└── pagination.md      ← 最初の 1 件、直接根

2 件目関連が来た:
ProdAI/
└── cursor_design.md   ← まだ根

3 件目で文件夹化:
ProdAI/
└── pagination/
    ├── README.md
    ├── offset.md
    ├── cursor.md
    └── infinite_scroll.md
```

事前に空フォルダを作らない。

---

## 3. 収録内容の標準（Quality Bar）

ProdAI に書き込む全 .md ファイルは下記基準を満たすこと:

### 3.1 必須メタデータ（ファイル冒頭）

```yaml
---
topic: pagination
applies_to: [frontend, backend, sql]
data_scale: [medium, large]
decision: when N > 100k → cursor; when frequent mutation → offset
status: stable
last_reviewed: 2026-05-19
---
```

### 3.2 結論先出し

文章は **適用条件 → 結論 → 根拠** の順:

```
## いつ使うか
データ量 > 100k かつ append-only な場合

## 結論
Cursor pagination + 時系列固定 ORDER BY

## 根拠
- offset は深分頁が O(N)
- cursor は O(log N + limit) で安定
- ...
```

### 3.3 落とし穴・反パターン必須

「この方案を使うときの罠」「失敗例」セクションを必ず含める。
AI が誤適用しないための安全網。

### 3.4 言い切る・濁さない

「〜と思います」「〜の場合もある」を避け、「A の時は B」「ただし C なら D」と明確に。

### 3.5 全 example が伪代码（§5 詳細）

特定言語コードを書かない。AI が読んで任意言語に翻訳できる伪代码のみ。

### 3.6 GitHub 高star 参考（必須・各「方式」≥1 件）⭐

各「方式 / 形式 / pattern」は **GitHub star > 5,000 の OSS リポジトリを最低 1 件**
明記すること。複数あれば併記。

- **闭门造车（憶測・自前判断のみで方案を作る）を禁止**。前期の調査・収集段階から
  高star OSS を一次情報源とする。「業界はこうしている」と書く前に、実在する高star
  実装で裏を取る。
- 各 doc に `## References` セクション必須。1 行 1 件、形式:
  ```
  - [owner/repo](https://github.com/owner/repo) — ~98k★ (verified 2026-05-23): 一言用途
  ```
- star 数は変動する。**必ず GitHub API / 公式ページで実測**してから記載し、
  `verified <日付>` を併記する。**捏造 URL・推測 star は §1 と同格の重大違反**。
- 閾値 5,000 を下回った参考は次回 review 時に、より高star な代替へ差し替える。
- 5,000★ 超の OSS 実装がそもそも存在しない方案は、ProdAI 収録基準を満たさない
  （= まだ「生産級の業界標準」ではない）。収録を見送るか、user に相談する。

---

## 4. 触发时机

以下の状況で AI は ProdAI への追加を **主動的に提案** する:

1. **重複 pattern 発見** — 「これ前にも同じパターン遭遇した」
2. **抽象化可能な実装** — 「この実装は汎用テンプレ化できる」
3. **教訓・決定の保存** — 「未来の AI / 開発者に伝えるべき洞察」
4. **業界調査・比較** — OSS ライブラリ比較、公開ベンチマーク等
5. **設計判断のフレームワーク** — 「offset vs cursor をどう選ぶか」等

提案形式: 「これは ProdAI 候補（〇〇 pattern）です。抽象化して追加しますか?」

---

## 5. 代码 = 伪代码（厳格遵守）

### 5.1 なぜ伪代码か

- ProdAI は **AI-first**。AI は伪代码を読んで任意言語に翻訳できる
- 特定言語コード（React/Rust/SQL）は version 変化で陳腐化する
- 伪代码は **長期保存可能**

### 5.2 伪代码の書き方

```
# CursorList - cursor pagination 用汎用テーブル
function CursorList<T>(props):
  state pages: list of list of T = []
  state pageIdx: int = 0
  state nextCursor: optional string = null
  state searchInput: string = ""
  state committedQ: string = ""

  effect on (deps change | committedQ change):
    seq = ++seqRef
    set isLoading true
    try:
      result = await fetcher({ q: committedQ, cursor: null, limit: pageSize })
      if seq mismatch: return
      pages = [result.items]
      pageIdx = 0
      nextCursor = result.nextCursor
    catch err:
      if seq mismatch: return
      onError(err)
      pages = []
    finally:
      set isLoading false

  function handleNext():
    if pageIdx < length(pages) - 1:
      pageIdx++ (cache から再表示、refetch なし)
      return
    if not nextCursor:
      return  # 最終ページ
    result = await fetcher({ q, cursor: nextCursor, limit })
    pages.append(result.items)
    pageIdx++
    nextCursor = result.nextCursor

  function handlePrev():
    if pageIdx > 0:
      pageIdx--

  render:
    toolbar with searchbox (Enter / button to commit)
    table with columns
    footer with prev/next nav
```

### 5.3 伪代码で NG

- `useState` / `useEffect`（React 特有）
- `async fn ...` の Rust 構文
- `SELECT ... FROM ... WHERE` の SQL 具体構文（必要な場合 SQL 風だが言語非依存に）

### 5.4 伪代码で OK

- `state X: type`
- `effect on (deps)`
- `function`, `class`, `interface`
- `try / catch / finally`
- `list of T`, `optional T`, `map of K to V`
- 業務無関係な変数名（`items`, `pages`, `cursor`, `query`）

---

## 6. 書き込みフロー

1. **AI 提案** — 「これを ProdAI に追加します（path: `xxx.md`）」
2. **業務情報除去確認** — 業務名・識別子を grep して完全除去
3. **伪代码化** — 特定言語コードを伪代码に変換
4. **GitHub 高star 参考の実測・添付** — §3.6 に従い、各方式に star > 5,000 の OSS を
   GitHub API 等で**実測**し `## References` に明記（`verified <日付>` 付き）。捏造禁止。
5. **User 確認** — 「OK ですか / 修正点ありますか」を必ず聞く
6. **書き込み実行**
7. **三語 README 同期** — 構造変更があれば README.md / README.ja.md / README.en.md を同時更新
8. **commit** — AI 自動可
9. **push** — §7 push 規則に従う

---

## 7. Push 規則

ProdAI の push は **必ず user に告知** すること。完全自動 push は NG。

許可動作:
- **commit** — AI 自動可
- **push to main** — 1 行告知（"main に push 完了 / これから push します"）後実行可
- **user 主導 push** — user が「自分で push する」と言えば AI は触らない

禁止動作（user 確認必須）:
- **force push** (`git push -f`)
- **branch delete** (`git push --delete`)
- **tag 作成 + push**
- **業務情報を含む可能性のある内容の push**（§1 §3 違反疑い時）

---

## 8. README 三語同期チェックリスト

README.md（中文）/ README.ja.md（日本語）/ README.en.md（English）を更新する時:

- [ ] 3 ファイル全て更新したか
- [ ] 目录构造（folder list）が一致しているか
- [ ] 新規追加ファイルが 3 言語に反映されたか
- [ ] バージョン / 日付情報が同期しているか

---

## 9. 違反時の措置

AI が誤って業務情報を持ち込んだ場合:

1. 即時停止
2. user に明確に報告（「業務情報 X が混入。削除します」）
3. 該当箇所を削除
4. push 済なら git filter-repo / BFG で履歴から除去（user の指示を仰ぐ）
5. 同じパターンの再発防止策を本ファイルに追記

---

## 10. このファイル自身の更新

`.claude/CLAUDE.md`（本ファイル）の更新ルール:

- AI が単独で書き換えない
- user の明示的な指示があった時のみ更新
- 更新時は本ファイルの「履歴」セクションに記録

### 履歴

- **2026-05-19**: 初版作成。ProdAI のセットアップ時に AI 行動規則を確立。
- **2026-05-23**: §3.6「GitHub 高star 参考（各方式 ≥1 件、star > 5,000、闭门造车禁止）」
  を追加。書き込みフロー §6 に star 実測ステップを挿入。既存 8 種 table 形式に
  実測済み References を遡及付与。
