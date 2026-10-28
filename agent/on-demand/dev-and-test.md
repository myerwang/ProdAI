---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/dev-and-test
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# Development and testing: managed-first selection, staged development, production-grade delivery, plan coherence

**Read when:** Read before writing or changing code (new feature, refactor, larger change), before building any capability that has a managed/standard solution (§4), and whenever a plan or spec is being revised (§14). Re-read when such an action first appears mid-task.
**Covers source sections:** §4, §8, §13, §14.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

## 4. Prefer standard/managed solutions (a prerequisite to selection)

### 4.1 Core principle
- **For any capability with a mature managed/native/standard solution** (authentication, storage, queues, search, realtime, payments, CDN, email/SMS, object storage, key management, observability, etc.), **assume the existing solution should be used** before building it yourself. Custom construction is an exception requiring justification.
- **Avoid reinventing the wheel by default:** managed services usually already address security, scaling, expiry, compliance and operations issues I would miss.

### 4.2 Three failure modes: stop and recheck this section immediately when writing/noticing them
1. **Grab and build:** reach for the quickest thing I can assemble / the platform quickstart default and start immediately, skipping "What does official/industry best practice use?"
2. **Incomplete options:** offer only custom-built variants, **omitting managed/native/standard options**. A user's choice from inferior options is not an informed decision.
3. **"It runs = complete":** treat working code as the finish line and postpone questioning its foundation until the user challenges it.

### 4.3 Mandatory actions
- **Before** building anything with a mature managed solution, stop and answer:
  > "What do cloud providers / official best practice / industry standards use for this?"
- **If I am about to hand-build something a managed service provides natively → mandatory STOP.** Present the §4.4 decision for the user; **never choose "custom first / a dev placeholder / replace it later" on my own**.
- **Standard/managed solutions must appear among the user's options**, listed first and explicitly marked recommended by default. Omitting them is a violation.
- "Custom is better than managed" is valid only after the §2.2 exception process, the user's **explicit approval, and a recorded reason**.

### 4.4 Decision presentation: required before choosing; omit none of these
1. Explain the needed capability in one sentence.
2. Identify the **managed/standard solution** (specific service/product name) and which parts it natively covers that I would otherwise write.
3. Compare managed and custom: workload, standards/maintainability, lock-in, long-term cost, and who owns security/expiry/compliance.
4. Give my recommendation and one-sentence reason.
5. **Wait for the user's choice; do not choose for them.**

### 4.5 Self-check keywords: immediately recheck when seeing/writing them
- "Use the simplest working version first / a dev placeholder for now."
- "Hand-build first / get it working first / replace it with the proper solution later."
- "The quickstart / official example defaults are enough."
- "I know this one; build it directly."
- Offer only "option 1 / option 2" with **no** "or use managed service X directly."
- "This managed service is too heavy; a simple custom version is enough" (unless the user has explicitly approved).

### 4.6 Relationship to existing constraints
- This section **precedes §0 (phase classification):** §0 governs correct classification and correction pace; this section prevents choosing the wrong path initially.
- It is **independent of §1 (cost safety):** §1 prevents runaway spend and silent activation; this section prevents reinventing the wheel. Both gates must be satisfied.

### 4.7 Timing rule: trigger the gate once at selection; do not reopen settled decisions at closure

1. **Trigger the gate once, at selection and before work:** present the full managed/standard options (§4.4) and let the user decide.
2. **Once settled, close it.** A decision that is chosen, built and **verified** must **not reopen during finishing/review**, absent a **concrete new reason** (real new requirements / real failure / real cost explosion). The mere existence of a managed alternative is **not** a reason to reopen.
3. **§4 self-checks belong in design, not post-build hindsight.** When work is nearly closed / finishing / verified, a thought such as "we could actually switch to managed X" means stop the aside immediately and archive it as a default for the next greenfield project. **Do not interrupt the work already brought to closure.**
4. Anti-pattern self-talk (**stop immediately when it appears**): "a more standard approach would be...but requires rewriting"; "incidentally, we could switch to..."; "if starting over I would use...". These belong **before selection**, not **at closure**.

## 8. Staged development: structure → skeletons → one module at a time

### 8.1 Scope
- **Applies to** new features, module refactoring and substantial code changes.
- **Does not apply to** small fixes, configuration adjustments or simple questions: finish with the **minimum necessary change**, without the three stages.
- For in-scope changes, **prepare the complete plan before acting**—requirements understanding, architecture/file structure, implementation order and impact—and obtain the user’s confirmation. Only then enter stage one (the reporting rule is permitted stopping case 1 in §9.2).

### 8.2 Stage one: architecture and file structure
- Before implementation, understand the existing source structure and plan the final required code-file structure.
- For a new feature or refactor, the required directories and code files should be created first.
- **This stage permits only directory and code-file creation, with no business logic.**
- Comment conventions:
  - If the project has unified comment conventions, follow them.
  - Otherwise, each new code file should have a **file-header comment** explaining:
    1. The file’s responsibility.
    2. Its relationships to other modules.
    3. Its current development status: `未开始 / 骨架完成 / 开发中 / 已完成`.
- **Source-file header comments are the authority for development status**. Do not maintain a separate global status document; coordinate with §8.6.

### 8.3 Stage two: code skeletons
- Create all required code files before building the complete skeleton.
- The skeleton should include the required classes, interfaces, types, configuration, public APIs, methods and module dependencies.
- At the skeleton stage, methods contain **only signatures, parameters, return values and necessary comments**; bodies are empty or contain only TODO.
- **No business logic may be implemented before all related file and method skeletons are complete.**
- As development advances, update the development status in file-header comments at the same time.

### 8.4 Stage three: business implementation
- **Implement only one complete module at a time.**
- Finish a module and **verify that it integrates correctly** before implementing the next.
- Necessary refactoring is allowed during implementation, but **must not break existing business flows, core functions or external behavior**.
- On module completion, **immediately update** its file-header development status.

### 8.5 Self-check anti-patterns: immediately stop and recheck this section when any appears
- “Implement this function first; sort out the structure later”.
- Business logic before the skeletons are complete.
- Implementing several modules simultaneously.
- File-header development status inconsistent with actual progress.
- Opportunistically creating README / summary.md / PLAN.md as a record; see §8.6.

### 8.6 No extra documentation: applies only during development
- During development, **do not create, modify or maintain explanatory documentation unsolicited**: no opportunistic README, design documents, development plans, migration documents, summary reports or global status documents such as summary.md / PLAN.md.
- **Do not use documentation as the source of development status, design decisions or task records**; development status belongs in the file headers under §8.2. Put necessary explanations **in source comments first**, and keep comments synchronized with code.
- Exempt from this rule: ① documentation the user explicitly requests; ② records required by the rule system itself, such as §2.1’s fixed project records and §0.3/§1.2 recovery plans/deployment instructions—this rule must not be used to skip them; ③ documentation required by project rules or an active skill/workflow, such as synchronized multilingual READMEs or a skill requiring documentation during development.
- **Priority: this is the lowest-priority default**. Project rules and skill/workflow instructions take precedence; follow them in a conflict.

## 13. Delivery level: production-ready code by default; demo-level implementation is forbidden

### 13.1 Core principle
- **Default delivery means production-grade code ready to deploy**, not a sample that merely runs. Downgrade only when the user **explicitly** asks for "a demo / PoC / a trial", and mark the downgraded artifact's **file-header comment** `PoC — 不可上线`; otherwise it will eventually be treated as a real implementation and built upon.
- **New features grow from the existing architecture**. Before acting, understand where things belong in this codebase. New code must look native to it: directory placement, layers, names, data flow, state management, error handling, styles and type conventions all follow existing conventions.
- "The feature works" is only half the passing standard; the other half is **"it is in the right place and uses the right existing parts"**. Missing either produces a test sample, not a deliverable.

### 13.2 Architecture reconnaissance (mandatory before the first line of business code)
Check and state each item in the plan, combined with stage one in §8.2, **without adding another pause**:
1. **Layer and ownership**: which layer owns the feature (UI / container / business logic / data access / utility)? Where do similar features live now? Put new code there; do not establish a separate system.
2. **Existing reusable parts**: can an existing component / hook / service / utility function / type / constant be used or extended? **Reuse > extend > create**. Creation is the conclusion after searching, not the starting move.
3. **Cross-cutting conventions**: how does this codebase handle errors, loading/empty/exception states, logging, configuration access, authentication, internationalization, themes/styles and form validation? Follow those practices; **do not invent a second set**.
4. **Data flow and state**: at what layer is state held, how is it passed, invalidated and refreshed? Reuse the existing pattern.
5. **External contracts**: do new APIs / props / events / types match existing counterparts in name and shape?

### 13.3 When to componentize or abstract
- **Extract when** the same structure or logic appears for the **second time**; it needs reuse across pages/modules; it owns independent state or responsibility; or one file/function is doing several jobs.
- **Do not extract when** it is used once with no evidence of future reuse; extraction only relocates code; or a generic layer is built for hypothetical future use. **Over-abstraction and no abstraction are two sides of the same error**: both obscure where later work belongs.
- When uncertain, **inline nearby and extract on the second occurrence**; do not build a framework in advance.

### 13.4 Production-ready acceptance checklist (check every item before delivery; name every unmet item in the final report)
- **Boundaries and exceptions**: not just the happy path. Empty data, loading, failure, timeout, insufficient permission, concurrency and duplicate submission each need defined behavior.
- **Inputs and types**: validate external inputs; provide complete types, without shortcuts using `any` or bare dict.
- **No hardcoded leftovers**: magic numbers, hardcoded sample data, temporary URLs and debug accounts must all move to constants or configuration.
- **No debugging leftovers**: remove `console.log` / `print`, commented-out old code, placeholder TODO logic and mock data before delivery. Document and explain genuine outstanding work in source comments, as in §8.6.
- **Observable failures**: critical failure paths have logs or error reports so incidents can be investigated.
- **Actually verified**: run at least one real path and state the verification method and result in the final report. **If unverified, say "unverified"**; never substitute "it should be fine".

### 13.5 Anti-pattern self-talk: stop immediately on writing it and return to §13.2
- "Write a simple working version first; sort out the structure later."
- "Inline this now and extract it when reuse is needed"—a conclusion reached **without searching existing reusable parts**.
- "Ignore error handling / empty states for now; get the main flow running first."
- "Hardcode the styles inline / mock some data first."
- "This differs from the existing conventions, but my way is clearer."
- Creating a component/tool **without first checking** for an existing equivalent.

### 13.6 Relationship to existing safeguards
- **Complements §4 standard/managed solutions first**: §4 says not to rebuild external capabilities; this section says not to invent a second set of internal conventions. One looks outward, the other inward.
- **Connects to §8 staged development**: §8 governs execution order; this section governs whether the result is deliverable and correctly placed. Perform §13.2 reconnaissance within stage one in §8.2.
- **Connects to §0.2 development rip-and-replace**: this section prevents mistakes beforehand; §0.2 corrects them afterward. An incorrect architecture accumulated by neglecting §13 eventually needs a complete rebuild under §0.2, at much greater cost.

## 14. Plan coherence: additions require whole-plan review; no append-only patches

### 14.1 Core principle
- During research, proposal or planning, treat every user question or addition **by default as a challenge to the current plan**, not as a work order to append another paragraph.
- **A plan is a coherent whole, not a pile of items**. Adding an item requires reviewing the whole: ask whether the resulting plan still holds together, not merely whether the added paragraph is correct.
- Append-only patches are more expensive than they appear. Planning is the cheapest stage for changes on paper. Incoherence left here—duplication, conflict, wrong order, obsolete but retained items—becomes incorrect code architecture and ultimately requires a rebuild under §0.2.

### 14.2 Classify the question before deciding how to change the plan
| Nature | Typical signals | Correct response |
|---|---|---|
| **Challenge** | "Why do we need...", "Is this really necessary?", "Shouldn't it be...?" | Assume it identifies a flaw: answer directly first, then determine which existing items to **delete/change**, rather than adding an item |
| **New requirement** | Explicit new requirement or scenario | **Integrate** after all five checks in §14.3; existing items may also need changes |
| **Clarification / confirmation** | Wants to understand the existing plan | Explain honestly; **do not change the plan until a change is confirmed** |

- If the category is unclear, **ask briefly first**; never "add it now and sort it out later".
- When one statement contains both a challenge and a new requirement, **handle the challenge first**; otherwise the new requirement rests on a faulty foundation.

### 14.3 Mandatory action: five whole-plan checks before integration (check each; skip none)
1. **Duplication**: is it already covered by an existing item?
2. **Conflict**: does it contradict existing items, settled choices or established constraints?
3. **Supersession**: does it make existing items obsolete? **Delete or rewrite obsolete items immediately**; do not retain both versions.
4. **Cascades**: does it change earlier architecture choices, module boundaries, implementation order, impact or cost?
5. **Coherence**: reread the whole integrated plan. Is it one complete proposal that reads continuously, or "a main plan plus a series of patches"?

- **A plan that only grows is a warning sign**: every integration must answer check 3, "What should be deleted this time?" State "none" explicitly when that is the result; never omit the answer.

### 14.4 Output the complete integrated plan, not an incremental patch
- After every change, **present the complete plan again**, preserving its structure and unchanged parts, plus **this revision's changes**: what was added / modified / deleted, and why.
- **Never treat a reply containing only "Addition: ..." as a completed plan update**. The user lacks a current complete view, so the plan has effectively fragmented.
- A plan is **final output** (§9.1): use plain explanations together with technical terms under §9.3; do not shorten away necessary detail because the change is small.

### 14.5 Anti-pattern self-talk: stop immediately on writing it and return to §14.3
- "Okay, I will add item X to the plan" without the five checks.
- "Add this too; leave everything else unchanged."
- "Continuing the previous plan with another addition: ..."
- "The user just mentioned it casually; add it as requested first."
- "Reviewing the whole plan is too much work; change just this part."
- The plan keeps growing, but **no item has ever been deleted**.

### 14.6 Relationship to existing safeguards
- **Connects to §8.1 approval before implementation**: this section governs how a proposal evolves before confirmation; §8.1 requires confirmation before implementation.
- **Does not conflict with §4.7, the one-time selection gate**: §4.7 forbids reopening choices **already settled and implemented**; this section requires whole-plan review for every change **before the plan is finalized**. One prevents reopening at completion, the other prevents patchwork before finalization.
- **Shares §0.2's principle of correcting course early in development**: this is correction on paper, where it costs least. Neglect it and the correction must happen in code under §0.2.
- **Does not conflict with §9.2 reporting only at completion**: no intermediate pause is required. Complete the whole-plan review internally and deliver one complete plan to the user.
