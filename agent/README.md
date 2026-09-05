# agent

本機 `~/.claude/` 配下の**グローバル AI 行動規則・設定の複本（ミラー）**。

このディレクトリは**実際に有効な設定そのものではない**。実体は `~/.claude/` 側にあり、
ここに置くのはバージョン管理・レビュー・他マシンへの展開を目的とした**写し**である。
写しは**常に実際にデプロイされている内容と完全に一致**していなければならない。

## 対応表

| repo 内のパス | 本機での実体 |
|---|---|
| `agent/AGENTS.md` | `~/.claude/CLAUDE.md` および `~/.codex/AGENTS.md`（共通本文） |
| `agent/settings.json` | `~/.claude/settings.json` |

`~/.claude/CLAUDE.md` というファイル名は Claude Code が読む固定名のため変更できない。
repo 側はツール中立な `AGENTS.md` を正式名とする。

**repo 側はツール中立・マシン中立を保つ**。個別環境に属する設定は repo に写さない。

### AGENTS.md

`agent/AGENTS.md` には**共通本文（§0〜§16）のみ**を置き、各ツール固有の末尾ブロックは写さない：

- `~/.claude/CLAUDE.md` → 末尾に `@RTK.md`（インクルード指令）
- `~/.codex/AGENTS.md` → 末尾に `## RTK - Rust Token Killer（Codex CLI）` セクション

したがって共通本文の行数（現在 924 行）までが 3 者で byte 単位に同一であること。

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

# settings.json は hooks を除いて比較
diff <(jq 'del(.hooks)' ~/.claude/settings.json) <(jq 'del(.hooks)' agent/settings.json)
```

差分が出た場合は、**実体側（`~/.claude/`）を正**として本ディレクトリへ写す。
