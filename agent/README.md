---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: load the resident entry; read each domain file in full only when its trigger matches
status: stable
last_reviewed: 2026-10-28
---

# agent — 常駐入口とオンデマンド規則

このディレクトリはツール／マシンに依存しない**共通規則の正本**。`AGENTS.md` は常駐入口、`on-demand/` は条件に一致する操作の**前**に全文を読む規則。全19章とReferencesを保持し、通常のタスクに毎回全ファイルを読み込ませない。

元の中文全文は tag [`agents-baseline-20261028`](https://github.com/myerwang/ProdAI/blob/agents-baseline-20261028/agent/AGENTS.md)（commit `f301b766a499b8ccd0447b1e00b87166156fb663`）に保存済み。本文の§番号はその原文番号、E番号は常駐入口内の番号。

## 構成と読み込み条件

| repo内のファイル | 必読になる時／範囲 |
|---|---|
| `agent/AGENTS.md` | 常駐。E1の条件表を新しい操作が現れるたびに再確認。 |
| `agent/on-demand/dev-and-test.md` | コード変更、自建能力の選択、plan/spec修訂前（§4/8/13/14）。 |
| `agent/on-demand/deploy-and-rollback.md` | deploy/release/upload/publish、アカウント依存書込、生産変更、GUI検討前（§0.3/7/11）。 |
| `agent/on-demand/external-paid-check.md` | 新計費点、有料自動チェーン、配額／従量バックエンドの操作前（§1/17）。 |
| `agent/on-demand/workspace-cleanup.md` | build成功後、タスク一時物生成・cleanup・残存一時物の最終報告前（§15）。 |
| `agent/on-demand/git-and-collab.md` | commit/push/sync/upstream/tag、異常な作業状態、branch/worktree操作、個人情報、破壊的操作前（§3/10/16/18）。 |
| `agent/on-demand/subagents.md` | 委派／子代理／並列agent開始前（§12）。 |
| `agent/on-demand/core-fulltext.md` | core規則の曖昧さ、対照、編集時（前言、§0の0.3以外、§2/5/6/9、References）。全体監査は全7ファイル。 |

同じsessionで読んだ同一版は再読不要。新しい操作が途中で現れたらE1を再確認し、対応ファイルを先に読む。読めない／途中で切れた出力を「読み込み済み」と扱わない。

`C20261028-COST` は現在のuserによる意味の明確化：用量の闸門は外部有料サービスの課金消費を対象とし、通常の会話／開発token予算ではない。新計費点、外部有料API／モデル、配額、retry、自動化の独立した保護は維持。

## 本機への配置

| 共通ファイル | Claude Code | Codex |
|---|---|---|
| `agent/AGENTS.md` | `~/.claude/CLAUDE.md` の共通部分 | `~/.codex/AGENTS.md` の共通部分 |
| `agent/on-demand/*.md` | `~/.claude/on-demand/*.md` | `~/.codex/on-demand/*.md` |

`on-demand/` は入口と同じディレクトリに置く。`rules/` のような自動読込ディレクトリへ移さない。マシン固有の追加設定はこのrepoに入れず、既存のローカル末尾や別設定を退避・保持してから共通部分だけ同期する。設定キーや認証／権限はこの文書同期で変更しない。

## 同期確認

入口と7ファイルを**個別に**比較する。下は共通部分の長さをrepoから得る例（既存のローカル追加は別途確認する）：

```bash
BYTES=$(wc -c < agent/AGENTS.md)
cmp agent/AGENTS.md <(head -c "$BYTES" ~/.codex/AGENTS.md)
for file in agent/on-demand/*.md; do
  cmp "$file" "$HOME/.codex/on-demand/$(basename "$file")"
done
```

同期方向はuserが指定した対象に従う。書込直前に元ファイルのhashを再確認し、同時編集を上書きしない。共通部分と既存ローカル追加を別々に照合する。既存sessionの即時再読込は保証しない。

## settings.json（今回変更なし）

`agent/settings.json` は共通設定の複本。`~/.claude/settings.json` のマシン固有`hooks`は写さず、比較は`jq 'del(.hooks)'`で行う。この規則の配布に伴って実体のsettingsやhooksを変更しない。

## 回復

変更前の共通入口は固定tagから読める。オンデマンド版から戻す場合は対象commit差分とローカル退避内容を確認して、userの指示範囲だけ復元する。既存tagの移動／forceや他のローカル追加削除を回復手順に含めない。
