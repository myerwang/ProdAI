# agent

本機 `~/.claude/` 配下の**グローバル AI 行動規則・設定の複本（ミラー）**。

このディレクトリは**実際に有効な設定そのものではない**。実体は `~/.claude/` 側にあり、
ここに置くのはバージョン管理・レビュー・他マシンへの展開を目的とした**写し**である。
写しは**常に実際にデプロイされている内容と完全に一致**していなければならない。

## 対応表

| repo 内のパス | 本機での実体 |
|---|---|
| `agent/AGENTS.md` | `~/.claude/CLAUDE.md` |
| `agent/settings.json` | `~/.claude/settings.json` |

`~/.claude/CLAUDE.md` というファイル名は Claude Code が読む固定名のため変更できない。
repo 側はツール中立な `AGENTS.md` を正式名とする（内容は byte 単位で同一）。

## 同期の確認

```bash
diff ~/.claude/CLAUDE.md    agent/AGENTS.md
diff ~/.claude/settings.json agent/settings.json
```

差分が出た場合は、**実体側（`~/.claude/`）を正**として本ディレクトリへ写す。
