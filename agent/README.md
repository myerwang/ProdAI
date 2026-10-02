# agent

本機の Claude Code / Codex / OpenCode の**グローバル AI 行動規則・設定の複本（ミラー）**。

このディレクトリは**実際に有効な設定そのものではない**。実体は各ツールのグローバル設定にあり、
ここに置くのはバージョン管理・レビュー・他マシンへの展開を目的とした**写し**である。
写しは**常に実際にデプロイされている内容と完全に一致**していなければならない。

## 対応表

| repo 内のパス | 本機での実体 |
|---|---|
| `agent/AGENTS.md` | `~/.claude/CLAUDE.md`、`~/.codex/AGENTS.md`、`~/.config/opencode/AGENTS.md`（共通本文） |
| `agent/settings.json` | `~/.claude/settings.json` |

`~/.claude/CLAUDE.md` というファイル名は Claude Code が読む固定名のため変更できない。
repo 側はツール中立な `AGENTS.md` を正式名とする。

**repo 側はツール中立・マシン中立を保つ**。個別環境に属する設定は repo に写さない。

### AGENTS.md

`agent/AGENTS.md` には**共通本文（metadata・全共通節・References）のみ**を置き、
Claude Code / Codex の各ツール固有の末尾ブロックは写さない：

- `~/.claude/CLAUDE.md` → `@RTK.md`（インクルード指令）および CodeGraph ブロック
- `~/.codex/AGENTS.md` → CodeGraph ブロック
- `~/.config/opencode/AGENTS.md` → 専用末尾ブロックなし

共通本文は repo と各ツールの実体で byte 単位に一致させる。行数は固定せず、repo 本文の現在の長さから求める。
同期前に既存の実体を退避し、共通本文だけを置換する。末尾は読み取った byte 列をそのまま保存し、
RTK / CodeGraph を消したり、別ツールの末尾をコピーしたりしない。OpenCode の専用グローバル入口は
`~/.config/opencode/AGENTS.md`。このファイルが存在すると OpenCode では Claude 互換の
`~/.claude/CLAUDE.md` より優先されるため、必ず同じ共通本文へ同期する。

§12 は発起時のモデル・推論強度の自動選択、§0/§18 は未宣言時の開発期・main 開発・分岐の退出規則を定義する。
これは指示の同期であり、存在しない「自動ルーティング設定キー」を config に追加するものではない。

### settings.json

`agent/settings.json` には**全マシン共通の設定のみ**を置く。以下は**個別（マシンローカル）設定**
として実体側 `~/.claude/settings.json` にのみ存在し、repo には写さない：

- `hooks`（`rtk hook claude` などツール／マシン依存の hook 一式）

したがって `hooks` 以外のキーが 2 者で一致していること。

## 同期の確認

```bash
LINES=$(wc -l < agent/AGENTS.md)
diff <(head -$LINES ~/.claude/CLAUDE.md)  agent/AGENTS.md
diff <(head -$LINES ~/.codex/AGENTS.md)   agent/AGENTS.md
diff ~/.config/opencode/AGENTS.md agent/AGENTS.md

# settings.json は hooks を除いて比較
diff <(jq 'del(.hooks)' ~/.claude/settings.json) <(jq 'del(.hooks)' agent/settings.json)
```

同期方向は user が指定した作業対象に従う。グローバル設定の収集なら実体 → repo、
本ファイルの編集とローカル同期を依頼された場合は repo → 両ツールの実体。
方向が不明な差分は先に比較し、古い側で新しい変更を上書きしない。
書き込み後は共通本文と末尾を別々に照合する。既存セッションへの即時再読込は保証されないため、
恒久的な反映確認は次の新規セッションで行う。
