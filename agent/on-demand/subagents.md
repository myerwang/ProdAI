---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/subagents
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# Subagents: when to delegate, model and effort routing, serial execution

**Read when:** Read before launching any subagent or delegating work, and before any parallel agents.
**Covers source sections:** §12.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

## 12. Subagents: use when needed, route intelligently, run serially by default

### 12.1 Authorization and responsibilities
- **Do the work yourself by default**: do not start a subagent merely to apply the routing table when direct file reading, search or commands suffice. Delegate only when the user requests it or applicable project/skill instructions explicitly require it, using the tool's supported permissions.
- **Once delegation is chosen, automatically match role, model and reasoning effort**; do not make the user choose a model every time or automatically inherit the parent's expensive model/highest effort. Follow the user's explicit model, effort or specialist-agent choice first.
- **The parent performs intelligent routing directly**: do not start another model to choose the subagent's model, and do not build a routing service. "Intelligent agent" means task-matching behavior; it does not imply a tool or model named `smart` / `auto`.
- Without explicit authorization in the current request, **at most one subagent may run at a time**, and subagents may not delegate further (recursion depth 1). Automatic model selection grants no extra concurrency and no authority to create branches, worktrees or new independent user tasks.

### 12.2 Establish all of the following before dispatch
1. **Scope**: explain why delegation is worthwhile (user request, independent specialist review, or bounded context isolation); list inputs, outputs, acceptance criteria and owned files. The parent handles work that needs its full current context or would compete to write the same file.
2. **Capabilities**: read the roles, callable models, supported efforts and permissions exposed by the current tools. Reuse that information within the session; refresh only when capabilities change or a model is unavailable. Never invent model IDs from memory or treat an API model list as the current client's available list.
3. **Routing**: under §12.3, choose the lowest sufficient tier, considering ambiguity, dependency span, failure consequences, tool/image capabilities and context capacity. A short task description does not make the task simple.
4. **Limits**: confirm the running count, attempt count and stopping conditions. By default, each subtask gets at most **2 attempts** (initial attempt + one evidence-based retry/upgrade), each lasting at most **10 minutes**. On timeout, stop first and collect completed results. A tighter user budget wins. Split longer work into bounded units with explicit deliverables; never use splitting to retry the same goal repeatedly.
5. **Visibility**: before dispatch, give one line: "subtask → role / model / effort / selection reason", then execute. This explains execution, not a new approval request. New billing points or budget overruns still follow §1 and §17.

### 12.3 Routing tasks to model tiers and effort

Choose capability tier first, then reasoning effort. **Higher effort cannot replace missing tools, context capacity or expertise.** These are behavioral defaults, not performance guarantees; exact model IDs and effort levels require the current-capability check in §12.2.

| Task characteristics | Model tier | Initial effort | Role/output requirements |
|---|---|---|---|
| Clear extraction, classification, formatting, narrow lookup | Lightweight, fast | low | Read-only extraction or explorer; return locations and evidence |
| Bounded code exploration, cross-file tracing, routine implementation or repair | Balanced | medium | explorer / worker; implementation needs file ownership and verification |
| Complex state/concurrency problems, cross-module root causes, independent review of substantial changes | Strong capability | high | Specialist analysis or reviewer; cover boundaries and counterexamples |
| Architectural ambiguity, security/production-critical problems, or demonstrated reasoning failure at a lower tier | Strongest currently available suitable model | high; upgrade to xhigh only with evidence | Return evidence, assumptions, risks and reproducible verification |

- **Codex name examples (verified 2026-09-11)**: lightweight `gpt-5.6-luna`; balanced `gpt-5.6-terra` / `gpt-5.6-sol`; complex tasks `gpt-6-astra`. Within a tier, choose using current tool descriptions, required capabilities and known costs; do not repeatedly trial same-tier models for comparison. When names are outdated or unavailable, remap by capability instead of copying historical names.
- Other clients use their own currently published model tiers and effort levels. If no effort parameter exists, select only the model and disclose that limitation; do not impose Codex enums on other clients. **Role and model are separate dimensions**: choosing explorer does not automatically choose a lightweight model.
- Use `max` / `ultra` only when explicitly requested by the user, or when an existing budget authorization and verification evidence justify it. Do not assign maximum effort to every subagent merely because the parent uses it. An upgrade still counts against the attempt limit in §12.2.

### 12.4 Parameters must take effect; failures must be disclosed
- When the tool supports explicit model and effort parameters, pass both, using a context-inheritance mode that permits overriding them; pass only necessary background and file locations. If full-history inheritance forbids model overrides, never claim to have switched models while inheriting the full history.
- When the client selects models through native agent configuration, reuse a suitable existing configuration first. A fixed model/effort in that configuration may override invocation parameters: check actual precedence before dispatch. Never automatically select a role marked "explicit use only".
- If the current tool cannot choose model/effort, the parent completes locally suitable tasks itself. If delegation is necessary, disclose the inheritance-only limitation; do not launch an extra CLI/API to bypass permissions, billing or concurrency constraints. Do not unilaterally replace an unavailable combination specified by the user.
- After dispatch, check the model/effort returned by the tool or available configuration records. If the platform does not echo them, report only "requested; platform did not echo"; never present requested values as observed actual values.
- Classify failure first: supply missing information; repair tools or report a blocker for tool/permission failures; upgrade the model or effort only for reasoning/capability insufficiency. Do not keep switching models for the same error or launch a replacement while the old agent still runs. After two attempts, the parent takes over or reports the real blocker.

### 12.5 Serial execution, acceptance and cost constraints
- Use foreground execution when supported. If only asynchronous execution exists, record the returned agent ID and wait for completion or confirmed stopping before starting the next agent. Wait intervals must not exceed **60 seconds**, so user instructions can be received. Neither a timeout nor an interruption request proves that the agent has stopped.
- Writing tasks need explicit file ownership. Tell subagents that others collaborate and that they must not revert others' changes. The parent centrally owns branches, commits, push and final integration. Read-only subtasks create no worktree; development writing subtasks also follow main's single-writer rule (§18).
- Accept based on evidence and actual results; a strong model does not waive verification. Reuse completed results so the parent does not repeat an entire scan.
- Enable concurrency only after explicit authorization in the current request. Count must not exceed the smaller of the user's authorization and the platform limit; when no count is given, retain a limit of 1. Before approval, explain count, responsibilities, budget and stopping method under §2.2. Do not automatically increase recursion depth or attempt count.

### 12.6 Anti-patterns and related rules
- Forbidden: "every subagent inherits the parent's highest effort"; "start an agent to decide which agent to use"; "just use auto as the model name".
- Forbidden: "retry the cheap model indefinitely"; "intelligent routing allows parallelism by default"; "automatically create a branch for every agent".
- §1/§17 govern cost and volume, §18 branches and the single writer, and §9.2 complete delivery. Routing exempts none of them.
- References at the end give sources for native implementations and model/effort configuration. This routing table expresses this policy's task tradeoffs; it does not claim that official documentation mandates a particular model for each task.
