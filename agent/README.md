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

**repo 側はツール中立を保つ**ため、`agent/AGENTS.md` には**共通本文（§0〜§12）のみ**を置く。
各ツール固有の末尾ブロックは repo に写さない：

- `~/.claude/CLAUDE.md` → 末尾に `@RTK.md`（インクルード指令）
- `~/.codex/AGENTS.md` → 末尾に `## RTK - Rust Token Killer（Codex CLI）` セクション

したがって共通本文の行数（現在 521 行）までが 3 者で byte 単位に同一であること。

## 同期の確認

```bash
LINES=$(wc -l < agent/AGENTS.md)
diff <(head -$LINES ~/.claude/CLAUDE.md)  agent/AGENTS.md
diff <(head -$LINES ~/.codex/AGENTS.md)   agent/AGENTS.md
diff ~/.claude/settings.json agent/settings.json
```

差分が出た場合は、**実体側（`~/.claude/`）を正**として本ディレクトリへ写す。
