# claude-config

`~/.claude/` 配下のグローバル Claude Code 設定の同期先。

## 同期対象

| repo 内のパス | 本機での実体 |
|---|---|
| `claude-config/CLAUDE.md` | `~/.claude/CLAUDE.md`（symlink） |
| `claude-config/settings.json` | `~/.claude/settings.json`（symlink） |

## 同期方式

- 本機の `~/.claude/CLAUDE.md` / `~/.claude/settings.json` は本リポジトリ配下のファイルへの **symlink**。
- `~/.claude/hooks/auto-sync-config.sh` が Claude Code の `PostToolUse`（Edit/Write）hook から呼ばれ、
  `claude-config/` に差分があれば自動で commit + push（main 直 push、分叉なし）。
- 詳細は `~/.claude/CLAUDE.md` §6 §7（書き入れた経緯）参照。

## 新しいマシンへの展開

```bash
git clone https://github.com/myerwang/ProdAI ~/ProdAI
# 既存 ~/.claude/CLAUDE.md / settings.json を退避してから:
ln -s ~/ProdAI/claude-config/CLAUDE.md ~/.claude/CLAUDE.md
ln -s ~/ProdAI/claude-config/settings.json ~/.claude/settings.json
# auto-sync スクリプトは settings.json から呼ばれるパスに置く:
#   ~/.claude/hooks/auto-sync-config.sh  （手動配置）
```
