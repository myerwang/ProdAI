---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/git-and-collab
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# Git and collaboration: identity, destructive and outbound operations, workspace sync, branch governance

**Read when:** Read before any commit, push, synchronization, upstream configuration, or tag creation/push; at session start when the workspace is not clean, not on main, has worktrees or lacks main; before the first commit of a new repository or the first push of a public one; before any force push, history rewrite or remote-branch deletion; whenever a branch or worktree must be created or cleaned; when personal information is involved (§3); and before any destructive local operation (§10.1).
**Covers source sections:** §3, §10, §16, §18.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

## 3. Personal and identity information

- **Do not use any user personal information without their explicit permission for this use** (email / real name / company / phone / address / IDs / employment information). Do not use it as Git author/committer, transmit it with commits/pushes, use it for messages or API parameters, or write it into any potentially versioned file.
- **Visibility is not permission:** personal information observable in injected context / past conversations / environment variables / git config / configuration files only makes it known. Each use is a separate decision requiring explicit permission; apparent convenience or reasonableness is not permission.
- **Git author identity must have a clear source:** before a new repository's first commit, ask which author email/name to use. In an existing repository, inspect `git config --local user.email`; reuse a noreply address directly. **If it is not noreply (a real email), stop and ask to replace it with noreply before committing.** Do not reuse it merely because history did so: that violates the first rule, and precedent is not authorization. Never guess. Personal information in a public push is effectively irretrievable.
- **Always use the hosting platform's noreply Git author address, never a real email.** GitHub uses `<user-id>+<login>@users.noreply.github.com`; get the user ID with `gh api user --jq .id`, never guess. GitLab and other hosts have their own noreply forms. Commit history is **effectively irreversible**, and crawlers index exposed real emails. Noreply correctly attributes commits to the platform account without exposing a deliverable address. This is the **platform's own standard practice**, not a workaround (the same principle as §4).
- **Apply this equally to public and private repositories; it is mandatory in private repositories too.** Private does not mean safe: a repository may become public, be forked/exported/migrated, and collaborators and the platform already see the complete author. Fixing it later requires rewriting the entire history and force pushing (§10.2), at far greater cost than configuring it correctly initially. **Do not branch on public/private status; always use noreply.**
- **Configure authors at repository scope by default** (`git config --local`); **do not change global `~/.gitconfig`**. Different remotes (company GitLab / self-hosted Git) require different authors; global changes spill into other projects.
- **Configure this before the new repository's first commit**, not "commit now and fix later"; as above, later repair is extremely costly.
- **A noreply address is public platform metadata, not personal information under this section.** It may be recorded in the project's `## 账号绑定` (§2.1 / §7.2), meeting both unique-ID binding and the prohibition on versioning real emails.
- If misuse has occurred, **stop immediately and report what information was used where**. The user decides remediation (deletion / repository recreation / history cleanup).

## 10. Irreversible or difficult-to-undo operations

### 10.1 Destructive local operations
- **Before deleting or overwriting non-regenerable content, obtain confirmation or preserve a way back**: user-edited files, uncommitted working-tree changes, and files/data I did not create. Before acting, either ask the user or make recovery possible (`git stash`, a backup copy, or moving to Trash). Never permanently delete them directly.
- **Guard each dangerous command**: never run `rm -rf` directly on a path assembled from variables (an empty variable can be disastrous). `git reset --hard` / `git checkout -f` overwrite uncommitted changes: first use `git status` to verify a clean working tree or stash the changes. Export a backup before `DROP TABLE` or clearing data in a development database.
- **Inspect the target before deletion**: if its actual contents differ from expectations (more, newer, or not created by me), stop and ask the user; do not continue with the original plan.

### 10.2 Outbound Git operations (global defaults; project rules may tighten or relax them)
- **commit: may run automatically** (local and reversible). Exception: before the **first commit in a new repo**, confirm the author identity under §3.
- **push: give the user one line of notice, then execute without waiting for a reply** ("Will push to <repo>/<branch>"). This informs the user; it does not request approval, which is how §16 automatic synchronization works. Before a public repo's **first push**, confirm author identity under §3 and verify the account under §7. Before every subsequent push, still perform the identity comparison in §7.4; do not skip that single command.
- **Force push, rewriting remote history, or deleting remote branches requires explicit user confirmation**; "cleanup" alone does not authorize automatic execution. Reuse an explicitly approved branch list or an approved platform policy to automatically delete source branches after merge, within its original scope, without asking again for every item. Before agent-executed deletion, still verify the target and retention conditions under §18.4; platform automatic deletion remains subject to the approved protection policy.
- **Creating and pushing a tag also requires notice by default**. Exception: the anchor tag in a §0.3 recovery plan is part of an approved production change; execute it automatically with that change, without a separate confirmation.

## 16. Keep the Git workspace clean: automatically synchronize at fixed checkpoints; write commit messages in Chinese

### 16.1 Core principles
- **A clean workspace means**: no uncommitted changes from this task, the authorized target branch synchronized with its remote, and no unexplained task branches/worktrees left behind (§18). Check `git status` and `git rev-list --left-right --count HEAD...@{u}`: synchronization requires both counts to be 0; checking only ahead commits misses behind commits. If no upstream exists, verify the target before configuring it with `git push -u`. **Do not publish whichever inherited branch happens to be checked out as a new remote branch.** Preserve and report others' uncommitted changes under §16.5.
- **The workspace must be clean when a session ends.** I cannot know when that will happen—the user may close the window at any time or context may be compacted. Therefore, do not defer all commits until “the end”; proactively synchronize at every §16.2 checkpoint.
- Reason: local-only changes exist on one machine and in one session's memory. The next session encounters unexplained changes it cannot safely delete or use and must ask the user; the result may be lost work or unfinished work mixed into a later commit.

### 16.2 Synchronization checkpoints: act immediately, without waiting for the user
| Checkpoint | Action |
|---|---|
| **Session start** | First verify phase, branch, worktrees, and remote under §18.2; handle inherited work under §16.3. |
| **A verifiable unit is complete**—a module completed and verified under §8.4, or an independently verified fix | Immediately commit + push. |
| **Before the final report** | Synchronization is mandatory; include commit SHA and branch in `📂 改动明细` (§9.7). |
| **Before stopping to wait for the user**—the three §9.2 cases, including the 🛑 section | Synchronization is mandatory: the session may end here. |

- Synchronization = `git add <本次改动的文件>` → commit (§16.4) → verify the account under §7.4 → push, immediately after the §10.2 notice. `git add -A` is allowed only after checking `git status` and confirming that it contains no files prohibited from the repository (§16.5).
- **The phase determines the destination**: development commits and synchronization go to `main`; production uses the same short-lived branch from §18.3 and merges to trunk only after verification and merge authorization are complete. Do not automatically create wip/backup branches for unfinished work, session endings, or tool defaults.
- Necessary intermediate commits are allowed during development, but must identify the unfinished scope. If pushing main would trigger an unauthorized deployment/release, retain the local commit and report the blocked external action. **Do not evade this by secretly creating a branch, and do not trigger production deployment without authorization.**
- Follow project requirements for user-led push; still commit this task's locally committable changes. Account, identity, sensitive-content, and other-workspace protections outrank cleanliness. Never violate §§3, 7, 10, or 16.5 to satisfy synchronization.

### 16.3 Inherited work found at session start
1. **Inspect before acting** (§10.1): review `git status` and `git diff` file by file.
2. Credentials, personal information, business information, or files clearly unsuitable for the repository → **do not commit**; report them in the first reply and await the user's decision in the 🛑 section.
3. Leave others' unfinished work under §16.5 untouched. Confirmed takeover work among the remaining inherited changes goes in a **separate** commit, such as `chore: 整理上次会话遗留的未提交改动`, never mixed with this task's changes. If inherited development work is on a non-main branch, first converge under §18.2; do not publish that branch directly.
4. If the authorized target branch is locally ahead, push after §7.4 verification. If behind, fast-forward only with a clean workspace. If diverged, inspect and integrate the actual differences; no force push, hard reset, or automatic new branch.
5. State the result in one line in the first reply, for example: “发现上次遗留 N 个文件，已作为独立 commit <sha> 推送”.

### 16.4 Commit messages: Chinese
- **Both subject and body must be Chinese**. English type prefixes such as `feat:` / `fix:` / `docs:` / `chore:` / `refactor:`, including an optional `(范围)`, may remain for tooling; **everything after the colon must be Chinese**. Example: `docs(agent): 新增 §16 git 自动同步规则，commit 一律中文`.
- The one-line subject explains what changed; the body explains why and the impact. Split unrelated changes into separate commits.
- Preserve English trailers required by tools/platforms, such as `Co-Authored-By:`, unchanged; they are not a violation.
- If project `AGENTS.md` explicitly specifies another commit language, follow it (§5).

### 16.5 Never automatically synchronize these contents; inspect each `git status` entry before synchronization
- Credentials / tokens / private keys / `.env` (§§1, 3, 7); the user's real personal information (§3);
- Business-project content entering a knowledge-repository project—an absolute prohibition in the project rules;
- Clearly local artifacts: build outputs, caches, IDE configuration, `.DS_Store`. Put them in `.gitignore`, not the repository;
- Files the user has partially edited that are outside this task (§10.1, content not created by me): leave them untouched and report them. If any category above is found, exclude that file, synchronize the rest normally, and name the exclusion in the report.

### 16.6 Self-check keywords: immediately reread this section when writing or seeing them
- “commit everything together at the end” / “push after everything is finished”;
- “commit locally for now; push when the user asks” when neither the user nor project rules require this;
- “the change is too small for a separate commit”;
- Starting work directly despite a dirty `git status` at session start;
- A final report without a commit SHA.

### 16.7 Relationship to existing safeguards
- **§10.2, push notice**: automatic push here means execution immediately after notice; force push and other high-risk actions still need user confirmation under §10.2.
- **§7, account verification**: perform §7.4 before every push. Automation is no exemption.
- **§3, personal information**: the author email must already be a noreply address; otherwise handle it under §3 before committing.
- **§0.3, production**: the production anchor-tag process is unchanged. This section prevents dirty handoffs; it does not change the pace of production changes.

> **Current user clarification C20261028-COST:** The usage/allowance/volume gates concern billing consumption of external paid services. They are not a model-token budget and must not be applied to ordinary conversation or development token consumption. Do not turn §17’s 1%/10% into a blanket token limit. Independent external paid API/model approval, new billing-point, subscription, quota, retry and automation safeguards remain in force.

## 18. Branch governance: development uses only main; exception branches require an exit path

### 18.1 Unified approach: trunk-based development

| Project phase / authorization | Working method | Branch budget and closure |
|---|---|---|
| Undeclared phase or explicitly development | Develop, verify, commit, and synchronize on `main` in the project's original checkout. | Default new non-main branches = 0; new worktrees = 0. |
| Explicitly production | Keep trunk releasable; when isolation is needed, use one short-lived branch + PR (GitHub flow). | At most 1 open task branch per agent per project; clean it after verification and authorized merge. |
| User explicitly requests a development branch, worktree, or parallel isolation | Use only the explicitly approved branches/count and target trunk. | Agree on merge, retention, or deletion in the same authorization; do not expand permission yourself. |

- Do not introduce Git Flow's develop/release/long-lived-feature branch system. Approved production maintenance branches may remain with their purpose and support period recorded. These counts are this workflow's management limits, not technical Git limits.
- **Task count, conversation count, and subagent count do not determine branch count.** New tasks reuse main by default; continued tasks reuse the same authorized short-lived branch. Do not create another branch for a retry, a second review round, or a new session.
- Manage branches with native Git and the host's PRs, branch protection, and delete-after-merge features. Do not build a branch database, AI branch-relationship graph, or cleanup service. Git history preserves integrated work; a branch represents active work, not chat history, backup, or an archive directory.

### 18.2 Entry checks: establish trunk first; inspect only necessary information
1. Read the project phase and account bindings. Run `git status --short --branch`, `git branch --show-current`, and `git worktree list --porcelain` to verify the repository, branch, uncommitted changes, and actual writers. List one refs overview at session start; refresh only when refs change. Do not repeatedly feed complete history to the model.
2. Check the remote URL against the project's declaration. After account verification, update branch references with `git fetch --prune origin`, then compare local and remote branches. This removes vanished **remote-tracking references**, not actual remote branches; do not add `--prune-tags`. For a non-origin remote, use the project's declared name rather than guessing a default. Estimate large-repository transfer volume under §17.
3. In development, if already on main with a usable workspace, continue directly. On a non-main branch or detached HEAD, first inspect unique commits and the workspace. If there is no unique work and main has no other writer, switch to main. If unique work exists, preserve it recoverably first, then integrate only this task's authorized, verified content. Preserve and report unknown-origin or out-of-scope changes once; do not automatically merge/cherry-pick them.
4. If the existing default branch is not main, or local main is absent, first check whether the remote has main; if so, make local main track it. If main is absent from the entire repository, inspect CI, the default branch, protection rules, and deployment dependencies before migration, and present a concrete migration plan. Initialize new repositories as main directly. Do not arbitrarily create an unrelated main, change the remote default branch without authorization, or continue development on the old branch.
5. **One writer on main**: if another task/user is editing the checkout containing main, coordinate or wait serially. Do not seize the branch, overwrite the workspace, or bypass occupancy with forced worktree options. Read-only analysis may continue. Until a subagent finishes its assigned files, the parent and other tasks must not write them concurrently.

### 18.3 Exception branches: define how they end before creating them
- In development, a branch is allowed only when the user explicitly requests isolation. Protection rules rejecting direct push, task complexity, fear of breaking things, and a tool's default worktree creation do not automatically grant an exception. Report the actual platform limitation and let the user decide whether to change strategy or approve this exception.
- When production needs a branch, create or reuse a short-lived branch from the latest verified target trunk. By default, use the tool's required prefix—`codex/` for Codex—followed by a concise task name; a user-specified name takes precedence. Never stack a new branch on another unmerged task branch.
- Before creation, specify purpose, owner, target trunk, acceptance conditions, merge method, and deletion authority. Use the existing PR description; without a PR, use local `branch.<name>.description`. Put cross-session authorization in the project's `## 例外清单`; do not create a separate branch ledger file.
- One branch corresponds to one unfinished task. Reuse or close this agent's existing open branch before creating another; changing names does not bypass the limit. Authorization for parallel agents does not itself permit additional development branches: isolation requires explicit branch authorization.
- By default, integrate and clean up at the end of this task. Necessary retention must state its reason, remaining work, and next handling condition in the PR/exception record. No update for 7 days is only a review prompt, **not deletion evidence**. Do not create background automatic cleanup unless periodic checking was requested.

### 18.4 Closure: verify → integrate → check → clean up
1. Synchronize this task's work under §16. Production merge/deployment still requires §0.3 recovery preparation and existing approvals. Permission to push is not permission to deploy.
2. Before deleting a candidate branch, refresh the remote and confirm that its final tip's work is integrated into the target trunk; it is not trunk, protected, or a maintenance branch; no unfinished PR or other PR uses it as a base; no active task/worktree depends on it; and it has no new unique commits. Inspect same-named local and remote branches separately: **one being merged does not prove the other is merged**.
3. The commands below use `origin/main`; production projects substitute their declared target trunk. For ordinary merge/fast-forward, prove reachability with `git merge-base --is-ancestor <tip> origin/main`. **Squash/rebase changes commit IDs**: `--merged`, a PR's merged label, branch name, or age alone cannot prove safe deletion. Also compare the final source tip, the corresponding PR's merged head, target trunk, and content/patch equivalence; confirm that no work was added after merging. Uncertainty goes to user decision; do not repeat the entire task just to clean a branch.
4. A local branch from this task that is verified integrated and unoccupied may be cleaned with `git branch -d`. That command also checks the upstream and **cannot replace step 3's proof against the target trunk**. If it refuses, investigate instead of mechanically switching to `-D`. Remote-branch deletion and forced local deletion still require concrete authorization. Execute an existing batch authorization within its list rather than asking again for each item.
5. For PR-based projects, recommend obtaining one explicit authorization to enable the host's automatic deletion of source branches after merge. Configure through the official CLI/API and read back to verify, retaining protection rules. This setting usually affects the whole repository; never enable it automatically for the user. The local checks and cleanup above remain necessary: remote auto-deletion does not mean local branches are clean.
6. Remove only this task's completed, clean, unoccupied worktrees whose content is integrated. Use `git worktree remove`, without `--force`, never direct directory deletion. After removal, check branch refs and clean remaining branches under existing authorization. Before deletion, recheck that tip/occupancy has not changed. If it has, reclassify rather than executing the old list.
7. Finally report commit SHA, target branch, and counts of branches/worktrees added, deleted, and retained in this task. Explain the reason and destination for every retained item. Do not report only that code is complete and leave branches for the next session to analyze.

### 18.5 Historical backlog: one snapshot, batch classification, analyze only unresolved differences
- When the user requests backlog governance, first stop creating branches. Batch-list ref names, tip SHAs, tracking status, trunk reachability, worktree occupancy, and matching hosted PR status using native Git commands. Use main as the common reference. **Do not compare every branch pair, start a subagent per branch, or repeatedly read all commits.**
- First batch-filter reachable branches with `git for-each-ref` / `git branch --merged origin/main`. Classify the rest as “PR squash/rebase integrated, awaiting verification”, “unique work remains”, or “active/protected/maintenance branch”. Read patches only for unresolved differences. Matching titles, stale dates, or similar branch names do not prove equivalence.
- Before cleanup, create and verify a `git bundle` containing the candidate refs and an exact ref→SHA list in a persistent local backup location outside the repository. Back up dirty worktrees, untracked files, and Git LFS objects separately: the bundle does not include them and must not be reported as a complete backup. Do not push the backup, put it in a knowledge repository, or rely on reflog expiry. Report its location and executable restoration method.
- Submit one concrete keep/integrate/delete list for user approval, including tips, evidence, and backup locations. **Complaints about too many branches do not authorize deleting unknown work.** After list authorization, execute §18.4 in a batch. Skip and reassess items whose tip/occupancy changed before execution, while finishing the rest.
- This snapshot serves only this governance task. Next time, check whether refs changed; if not, reuse the findings. Do not expand an ordinary feature task into repository-wide branch archaeology. When historical backlog is found, ensure this task adds zero branches, report the count, and obtain separate scope authorization to clean existing branches.

### 18.6 Anti-patterns and related rules
- Never “create a branch for each new task”, “give each subagent a worktree”, or “automatically open wip because main rejects push”.
- Never “assign hundreds of agents to hundreds of branches”, “delete every branch with a temporary-looking name”, or “use -D when -d refuses after squash”.
- §0 determines phase, §12 governs delegation, and §16 governs synchronization. This section governs branch creation and exit; those workflows' default actions cannot override it.
- §7 identity verification and §10 recovery/deletion authorization still apply. Achieving zero new branches must not cost the loss of old work.
