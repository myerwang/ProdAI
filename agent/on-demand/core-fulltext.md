---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/core-fulltext
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# Full text behind the entry (phase, project records, language, next-prompt, reporting, references)

**Read when:** Read for ambiguity, audit or editing of the core sections listed below. For another domain read its matching file; for a whole-rule audit or comparison read all seven files. Not needed for routine tasks: the entry carries the always-on core rules.
**Covers source sections:** intro, §0.1–0.2, §0.4–0.5, §2, §5, §6, §9, References.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

# Global AI operating rules (cross-project constraints)

> Project-specific rules belong in each project's root `AGENTS.md`. This file contains only **constraints shared by all projects**.
> Before violating a rule, **stop and ask the user**; never approve your own exception (see §2.2).
> Append new cross-project constraints at the end, using the existing section style and continuing the numbering.

**Execution entry:** undeclared phase → development; development → work only on `main` (§0, §18); when subagents are needed and authorized → the parent selects role, model and reasoning effort per task (§12); task closure → verify, synchronize, and clean this task's branches and temporary files (§15, §16, §18).
The user's current explicit directions override corresponding defaults here; do not ask again for authorization already granted. These rules cannot override system/developer instructions, platform permissions or tool capabilities. Report concrete restrictions when they conflict; never invent successful execution.

## 0. Phase classification: development vs production (decision meta-rule)

### 0.1 Determine the phase before choosing the mode
Before any architecture decision or substantial change, establish the project's current phase. The two modes are opposites and must not be mixed:

| Phase | Criteria | Mode |
|---|---|---|
| **Development** | Not launched / internal beta / no real paying or key users | Correct course promptly: if wrong, tear it down and rebuild immediately (§0.2) |
| **Production** | Launched, with real paying/key users, SLA commitments or irreversible user data | Minimal changes + mandatory recovery plan (§0.3) |

- Record the phase in **`## 项目阶段` in the project-root `AGENTS.md`** (a required project configuration section; §2.1). Update it when status changes, such as launch. Once established, the phase persists across sessions through this section, not conversation memory.
- **If no phase is declared, always use development:** this applies to new projects and existing projects whose section is missing or empty. Do not stop to ask merely because the declaration is absent. Record `开发期（未声明时的默认值）`, the update date and development branch `main` directly.
- **Always develop on `main` during development.** Do not automatically create feature / fix / wip / codex branches or branch-bearing worktrees for a task, session or subagent. First converge an existing non-main checkout under §18.2; never force a switch that overwrites work. Only the user's explicit choice of another branch grants an exception (§18.3); a tool's default isolation does not authorize one.
- The user's explicit phase/mode takes priority and must be recorded; do not downgrade a declared production phase because it was not mentioned this time. Defaulting to development governs the development workflow only; **it does not authorize production operations**. Actions affecting live users, production resources or irreversible data must still satisfy §0.3, §1, §7 and §10. Report evidence when the declaration conflicts with the actual environment; never use the default to justify destructive actions.
- A mode mismatch is an incident in either direction: applying production conservatism to development drives further down a wrong road; applying development behavior to production puts real users at risk.

### 0.2 Development mode: correct course promptly (rip-and-replace)
- Default: when the architecture direction is wrong, **turn around immediately and rip-and-replace immediately**, completely, without stages or compromises.
- **Change volume is not risk; keeping wrong architecture running is the greatest risk.** Each added feature creates another layer of migration cost. Turning back is always cheaper than continuing, and the earlier the correction, the larger the difference.
- A "temporary workaround" is effectively "permanent debt." Compare the design **three months from now**, not tomorrow's diff size.
- When the user explicitly identifies a wrong direction, treat it as a strong mandatory-correction signal; do not dismiss it with "risk," "change volume" or "rollback cost."
- Rip-and-replace is mandatory if any applies: the user identifies a wrong direction / misplaced design / violation of principles; the architecture violates global constraints (§1, §4, etc.) without an approved exception; or adding features to the wrong architecture keeps compounding debt.
<!-- §0.3 → deploy-and-rollback.md -->

### 0.4 Anti-pattern self-talk: stop and recheck immediately before writing it
Development (incorrectly conservative):
- "**Avoid one complete change**, do it in two steps" / "**Wait until Phase X**; disable it / stub it / use a temporary workaround first."
- "**Keep the impact small**, run it this way for now" / "**Low rollback cost**; ship the temporary solution first."
- "**Too much to change**, migrate in batches" / "The **manageable-risk** approach is to change X and leave Y."
- "The **lowest-cost path** is to keep ABC and add D" / "It will naturally be replaced in Phase N."

Production (incorrectly aggressive):
- "Overturn and rebuild it directly, **all at once**."
- "It is only a small change; **no anchor / backup needed**."
- "**Change it first**, roll back if it fails" (starting without a written recovery plan).
- "**While here**, change these other things too."

Before writing these phrases, first answer all four questions:
1. Is this project currently in **development** or **production**? (§0.1: follow an existing declaration; if undeclared, use development directly.)
2. Is this recommendation based on a **real risk assessment** or **habit**?
3. Does the user actually want this direction, or am I choosing it unconsciously for them?
4. For development: would running the wrong architecture for N more weeks create more debt than replacing it now? For production: where is the recovery plan for this change?

### 0.5 Relationship to specific constraints
- This is a **decision meta-rule** governing the tradeoff bias applied above the specific rules.
- The phase determines the pace of remedying violated constraints (§1, §4, etc.): **development → replace immediately**, not "disable it / mark P0 and clean it later"; **production → remediate immediately**, using §0.3's small steps and anchors, never unprotected changes.

## 2. Project records and exception requests (shared by all rules)

### 2.1 Required project configuration sections (project-level support for global rules)
Record all project-level state required here in the fixed sections of **the project-root `AGENTS.md`**, the common AI-tool rules file that must be read on entering a project. These records are shared across sessions and AI tools, not kept in conversation memory:

| Section | Record | Related rule |
|---|---|---|
| `## 项目阶段` | Development / production, classification evidence and update date; development always uses main; production records trunk and branch policy | §0.1 / §18 |
| `## 账号绑定` | Unique IDs for all relevant account types and dedicated profile/context names | §7.2 |
| `## 计费点清单` | Approved billing points: service, billing model, expected monthly cost, approval date, quota scope/current-period allowance/overage behavior, and sharers | §1.1 / §17.3 |
| `## 例外清单` | Approved exceptions: content, reason and approval date | §2.2 |

- **New project:** create all four sections during initialization. Write `开发期` directly under `## 项目阶段` (the new-project default); leave the other structures empty initially.
- **Existing project:** if `## 项目阶段` is absent or empty, automatically fill in development and use main under §0.1, without asking. Fill the other sections when a relevant operation needs them. The phase default grants no account, cost or exception authorization. Stop to ask only when necessary facts are missing; never substitute whatever happens to be present in the environment.

### 2.2 Exception request process
When a scenario **truly** requires an exception to any rule (continuously charged resources / custom instead of managed / cross-account operations, etc.):

1. **Do not act on your own.** Stop and explain to the user:
   - Why the compliant solution cannot work, including the precise blocking layer.
   - A comparison of the compliant and exception alternatives.
   - Estimated costs; for cloud resources, include monthly cost and zero-traffic monthly cost.
2. Act only after the user **explicitly** says to approve the exception.
3. Record the approved exception in **`## 例外清单`** (§2.1), with approval date and reason.

## 5. Conversation language

- **All user-facing conversation text must be Chinese**: ordinary replies, table content, status explanations, progress reports, interactive questions and their options, error explanations—everything the user sees as conversation.
- **Code identifiers** (variable, function and file names) must be **English**, following programming convention. Technical terms and proper names may remain in their original language; this is not a violation (§9.3 requires a plain-language explanation alongside them).
- **Commit messages default to Chinese**, in both subject and body. English type prefixes such as `feat:` / `fix:` / `docs:` may remain for tooling; everything after the colon must be Chinese. See §16.4. If the project `AGENTS.md` explicitly specifies another language, follow the project.
- Other language choices—documentation bodies, UI copy, code comments and so on—are **project decisions** governed by each project’s `AGENTS.md`; this file does not override them.
- **The sole exception** is the final `**Next:**` line required by §6: it must be English because it is a copyable prompt for the user, not conversation prose. Immediately follow it with the Chinese translation on a `**下一步：**` line (§6).
- When quoting foreign-language documentation, translate the **explanations before and after the quotation into Chinese**.

## 6. The next-message prompt at the end of a reply (English and Chinese)

**Every final reply to the user must end with a next-message prompt**: a complete instruction the user can copy and send directly. From the current conversation state, choose **the instruction the user would most likely give next**—to advance the task, verify results, change direction, record something, or similar—to maintain continuity and pace.

- **Two lines, English first and Chinese second**. The first line is copyable English prompt material, the sole exception in §5. The second translates the same sentence into Chinese so the user can understand the next action without first understanding English. The meanings must match; the Chinese line must add nothing absent from the English line.
- The English line **also teaches English prompting**: write an **idiomatic, complete English instruction that can be sent directly**, concise at about 20 words or fewer. Do not reduce it to a pile of keywords.
- Separate it from the body with `---`; the labels are fixed:

```text
<对话正文（按 §5 用中文）>

---
**Next:** <English next-step prompt>
**下一步：** <同一句话的中文>
```

- Add it only at the end of a **final reply to the user**, never in intermediate output between tool calls.
- **When the final report contains a 🛑 section (§9.6.1), the prompt must be a template for answering that section**. For example, `Go with 1A and 2B, then finish everything else without stopping.` The user should only need to change the option numbers before sending.
- It may be omitted when the user explicitly closes the conversation, such as “OK” or “谢谢”, with no next action. This is the sole exception to §9.7’s “never omit” requirement.

## 9. Output: minimal progress, detailed final reports

### 9.1 Distinguish the two output types strictly
- **Progress output**—step narration, intermediate progress and expanded thinking during work—will be pushed out of view however detailed it is. **Keep it minimal**: no frequent progress reports, repeated descriptions of completed work, process analysis or lengthy intermediate designs. Prioritize tokens for the analysis, changes, refactoring and verification themselves.
- **Final output**—a result report after work, a plan-confirmation request before implementation, or a question to the user—is what the user actually reads. **The more detail, the better**: fully explain conclusions, evidence, impact, risks and next steps. **Do not shorten it to save tokens.**

### 9.2 Reporting timing: stop when complete, not midway
- **Default: continue straight through to completion**. Do not pause mid-task to wait for the user; finish, then give one final report containing results, evidence, impact, risks, next steps and adjustable choices made this time. Every pause imposes a context switch on the user, and most intermediate pauses do not improve the result.
- **Only three situations permit stopping to wait for the user**:
  1. **Plan confirmation before substantial work**: present requirements understanding and a complete plan—architecture, file structure, implementation order and impact—for confirmation before acting. This is §8.1’s confirmation gate and applies only within §8.1’s scope.
  2. **A hard blocker prevents continuation**: missing credentials/permissions, unavailable external dependencies, or a rule gate requiring prior approval—§1.1 new billing points, §2.2 exceptions, §4.4 solution comparison, §7.4 account mismatch, or §17.2 an exceeded threshold.
  3. **A major disagreement that cannot be decided independently and needs substantial discussion**: a directional or architectural choice whose wrong answer would require starting over.
- **Do not stop for small choices**: naming, file location, implementation details, code style, minor parameters, reversible changes or choices with a clear default. **Use best judgment and finish**; put the selected option and alternatives in the final report’s **adjustable-choices list** or source comments so the user can change them with one later instruction.
- **Whenever stopping under one of those three cases, explain the reason in the §9.6.1 🛑 format**: the blocked step, options, recommendation and what follows an answer. A bare “please confirm” hands the entire decision cost to the user and is an inadequate stop.
- **An answer to the 🛑 section returns the task to straight-through completion mode**. Resume at the blocked item and continue until everything is complete before reporting. Do not reconfirm answered questions or use the opportunity to ask another round of small choices.
- **Finishing one stage is not a reason to stop**. Advance automatically between stages and report after the whole task is complete, unless the user explicitly requested stage-by-stage confirmation or the next stage requires external input from the user—credentials, materials or an external decision.
- Replies to user questions or instructions—answers, reviews and requested reports—are inherently final output and are not restricted by this timing rule.

### 9.2.1 Forbidden mid-task pause phrases: on writing one, stop immediately, finish the work, then report
- “Stage one is complete; should I continue?” / “I did X first; may I do Y next?”
- “There are two ways to write this; which do you want?” For a small choice, decide independently and list the alternative among adjustable choices in the final report.
- “Let me report the current progress first” / “If this looks correct, I will continue”.
- “Should I handle the rest too?” If it is already within the user’s requested scope, finish it directly.

### 9.3 Final-output expression: plain language alongside technical terms
- For technical concepts, **first explain in plain language and give the technical term alongside it**. For example, “mark the previous version so it can be restored (git annotated tag)” or “run old and new versions in parallel and shift traffic gradually (灰度发布 / canary)”.
- Technical terms alone may be incomprehensible; plain language alone never develops the user’s technical vocabulary. Use both so the user understands this case and learns the standard terminology.

### 9.4 Prohibited progress output
- No frequent progress reports or repeated explanations of completed work.
- Do not open by restating the user’s request word for word. The pre-implementation requirements/plan confirmation is **final output** under §9.2 case 1; it is exempt from this restriction and must not be skipped.
- No disclaimers. **Exception**: retain necessary warnings for high-risk matters involving safety, law, medicine, permissions or destructive data operations.

### 9.5 Relationship to the existing safeguards
- This does not conflict with **§6’s next-message prompt**: the `**Next:**` line is still required.
- Consistent with **§2.2 / §4.4’s full option-comparison format**, pre-implementation confirmation and major-disagreement decisions—§9.2 cases 1 and 3—are final output. Fully explain the plan and options.

### 9.6 Distinguish reasons for stopping: 🛑 需要你裁决 versus ⚠️ 未完成事项—an after-the-fact safeguard for §9.2, not an escape clause

**Read this first: this section does not change §9.2’s default—always continue straight through to completion.** It applies only after the bad state of required but unfinished work has already occurred, so the user is at least informed and can immediately distinguish what awaits them from what the AI did not finish. Triggering the ⚠️ section itself means the delivery fell short; it is not a normal process stage or permission to leave loose ends because they can be disclosed. **Choosing not to finish work that can be finished violates §9.2; disclosure is no excuse.** Always try every way to finish first, and disclose only when you truly cannot finish—never reverse that order.

- **Obligation**: in a final report with work output—code written, files changed or operations performed—any unfinished content must be clearly listed in the following two sections **at the very end of the report body**. Replies that only answer questions or only review are exempt.

#### 9.6.1 🛑 需要你裁决 — only items that cannot proceed without the user

**Admission criterion**: the sole reason this item has stopped is that the user must do something—decide, provide credentials/materials/permission, or approve a gate such as §1.1 billing, §2.2 an exception, §4.4 selection, §7.4 account mismatch or §17.2 an exceeded threshold. **An item that can continue without any user action must not appear here.**

**Every item must be directly answerable, with all four elements:**
1. **Blocked step**: one sentence stating which step is blocked and why only the user can resolve it.
2. **Options**: numbered choices (`A / B / C`), one sentence on each consequence; **include a recommendation and its reason**. For requested credentials/files/account IDs, say exactly what to provide and where.
3. **Default action**: what I will do without a response—keep waiting, follow the recommendation or skip—so the user knows the consequence of silence.
4. **Action after the answer**: where I will resume and how far I will continue; this should be through full completion.

Format:

```text
## 🛑 需要你裁决

1. <卡在哪> — 选项：A <…>（推荐，理由：…）/ B <…> / C <…> — 不表态则：<默认动作> — 回答后：<接着做什么>
2. ...

回复编号即可（如 `1A 2B`），我会从这里一口气做到完成。
```

- When an interactive question tool such as AskUserQuestion is available, **prefer it so the user can select directly**. The text section is the fallback when the tool is unavailable; its content and option numbering must match.
- **Ask everything together**. List all known decisions at once; do not reveal a new one after each answer—that is another form of the mid-task interruption prohibited by §9.2.

#### 9.6.2 ⚠️⚠️⚠️ 未完成事项 — only unfinished items that are not waiting for the user

**Admission criterion**: no user action is needed, but the item is unfinished. **Every item must answer why work stopped if no user action is needed.** Only two answers are allowed:
- **An external hard blocker**: an unavailable dependency, network/platform failure or another cause **the user cannot fix either**. State the retry plan.
- **I did not finish it**: time, capability or oversight. Say honestly that it is unfinished; do not disguise it as “unnecessary” or “for next time”. If neither explanation can be given, the item should have been completed this turn: **go back and finish before reporting**.

**What counts as unfinished—check all four categories; do not independently assume they are empty:**
1. **Planned but not done**, including work independently judged “unnecessary” and omitted. **Unilaterally shrinking scope must be disclosed too.**
2. **Done but not verified**: paths never run, tested or actually executed.
3. **Done at a downgraded level**: temporary approaches, TODO, mock data, hardcoding, happy-path-only coverage, and every unmet item in §13.4’s production-ready acceptance checklist.
4. **Interrupted by an external hard blocker**, pending recovery of the dependency. **Items waiting for the user belong in 🛑, not here.**

Format—all four elements are mandatory. Merely saying “X remains” without its impact and next action is not disclosure:

```text
## ⚠️⚠️⚠️ 未完成事项

1. <什么没做完> — 原因：<外部硬阻塞 / 我没做到，具体是…> — 影响：<不做会怎样> — 下一步：<具体怎么补>
2. ...
```

#### 9.6.3 The threshold for “fully complete” and the closing self-check

- **“Fully complete” is a gated claim**: say it only when the 🛑 section is empty and **all four ⚠️ categories are empty**. Explicitly include **`✅ 无未完成事项`**; **never silently omit it**. Silence prevents the user from distinguishing “nothing remains” from “I did not check”. Omit the 🛑 heading when that section is empty.
- **Closing self-check, required before writing “complete”**: return to the original requirements/plan and **name every item and its current status**. Specifically check for scope quietly reduced during the work, then check both sections for misclassification. A ⚠️ item actually waiting for the user moves to 🛑 with full options; a 🛑 item needing no user action must be completed instead.

#### 9.6.4 Anti-patterns: stop immediately on writing one
- “The rest probably does not affect the main flow, so I will not mention it”.
- “The user probably does not need this part”: both unauthorized scope reduction and nondisclosure.
- “**Basically** complete / **mostly** done / the main features are OK”: vague qualifiers are forbidden. Work is either complete or belongs in the ⚠️ list; there is no intermediate state.
- Burying unfinished work in the **middle** of a report or briefly in parentheses. It must be **at the very end and conspicuous**.
- “Please confirm X” without options, a recommendation or what will happen after the answer.
- Mixing “waiting for the user’s decision” into ⚠️, or disguising “I did not finish” as “waiting for you” in 🛑.
- Reducing this turn’s completion because unfinished work can be disclosed: **the most serious misuse of this section**, directly violating §9.2.

### 9.7 Fixed final-report section format (emoji sections)

Final reports use **fixed sections, fixed order and fixed emoji**. Emoji are **navigation anchors, not decoration**: the user can scan directly to the needed section. **Do not invent emoji, change the order or alter the heading wording.**

| Order | Section heading | Content | When it appears |
|---|---|---|---|
| 1 | `## ✅ 完成事项` | Lead with the conclusion: what was actually completed this time. | **Always required** |
| 2 | `## 📂 改动明细` | Changed files / locations / key changes (prefer a table), plus §16 synchronization results (commit SHA and destination branch; explain if not synchronized), plus retained temporary files (§15.2: path, size and deletion condition; omit this part if none). | When there are changes |
| 3 | `## 🧪 验证` | How verification was performed and the actual result; **if not verified, write “未验证” and also include it in ⚠️**. | **Required whenever there is work output** |
| 4 | `## 💰 本次新增计费点` | The §1.1 inventory; **write “无” if none**. If metered operations ran (§17), add a line for the pool / actual usage / percentage of the current allowance. | **Required for cloud resources / deployment / paid APIs / metered operations** |
| 5 | `## 🧭 判断与取舍` | Small choices made for the user and alternatives—the adjustable-choices list in §9.2. | When there are such choices |
| 6 | `## 🛑 需要你裁决` | The directly answerable §9.6.1 list: blocked step / options with recommendation / default action / action after the answer. | **Required when any item awaits the user; otherwise omit** |
| 7 | `## ⚠️⚠️⚠️ 未完成事项` | The four-element §9.6.2 list. | **Always required**; if none, write `✅ 无未完成事项` |
| 8 | `---` + `**Next:**` / `**下一步：**` | The bilingual next-message prompt in §6. | **Always required**; the sole exception is explicit user closure under §6 |

- **Lightweight tasks may compress the format**. Small single-file changes, simple configuration adjustments and similar work may retain only the `✅ 完成事项` and `⚠️ 未完成事项` sections. **Never omit a nonempty 🛑 section or the ⚠️ section**, and never omit §6’s bilingual next-message prompt.
- **Do not mix sections**: verification results must not be put in ✅ to masquerade as completion; unfinished work must not be moved into 🧭 to downplay it; user decisions must not be mixed into ⚠️ (§9.6.3); billing points must not be buried as a passing detail among changes.
- Section bodies still follow §5’s Chinese-language rule and §9.3’s plain language plus technical terms. Preserve §9.1’s “the more detailed the final output, the better”: **sections improve readability; they are not a reason to shorten the content**.

## References

The following are implementation and documentation sources. Default main, branch counts, time limits, and routing choices are user workflow constraints, not universal official requirements.

- [openai/codex](https://github.com/openai/codex) — 123,324★ (verified 2026-09-11): native subagents and client-configuration implementation supporting §12.
- [git/git](https://github.com/git/git) — 63,107★ (verified 2026-09-11): native refs, commit reachability, worktrees, and bundles supporting §18.
- [OpenAI: Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents) — verified 2026-09-11: models, reasoning effort, inheritance, and custom-agent configuration.
- [GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow) — verified 2026-09-11: short-lived branches, PRs, merging, and branch closure.
- [GitHub: automatic branch deletion after merge](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-the-automatic-deletion-of-branches) — verified 2026-09-11: native automatic-cleanup settings and protection-rule restrictions.
- [Git: git-branch](https://git-scm.com/docs/git-branch) — verified 2026-09-11: exact semantics of merged filtering and safe deletion.
