---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/external-paid-check
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# External paid services: billing points, automated chains, metered operations

**Read when:** Read before creating any billing point (cloud resource, paid API/model, subscription/trial, tier or quota change), before deploying or running any automated chain that calls a paid service (including local test scripts), and before any metered operation on a quota-limited or usage-billed backend, including read-only, manual and one-off queries (§17). Re-read when such an action first appears mid-task.
**Covers source sections:** §1, §17.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

**User clarification C20261028-COST (2026-10-28, not in the tag):** The usage/allowance/volume gates concern billing consumption of external paid services. They are not a model-token budget and must not be applied to ordinary conversation or development token consumption. Do not turn §17’s 1%/10% into a blanket token limit. Independent external paid API/model approval, new billing-point, subscription, quota, retry and automation safeguards remain in force.

## 1. Cloud cost safety (prevent runaway bills)

Cloud misconfiguration can cause unbounded loss: a loop can turn usage charges into an enormous bill, and silent activation can make the user pay continuously without knowing. This section is a prerequisite for **every cost-incurring operation** (cloud resources, paid APIs, third-party SaaS, paid-tier/quota changes). Project rules define specific architecture preferences, such as serverless first.

### 1.1 Every new billing point: present the decision and obtain approval; never activate silently
- **Anything that can appear on a bill is a billing point:** a new cloud resource, paid API/model call, subscription/trial that converts to paid, tier upgrade or quota increase.
- Before activation, explain and obtain the user's **explicit approval** for the billing model (usage / availability), monthly cost at zero usage, expected normal monthly cost, and **runaway upper bound** (worst-case loop/retry-storm spend and whether a natural cap exists).
- **Free allowance does not waive disclosure:** charges begin when it runs out. "It is within the free allowance, so no need to say" is silent activation.
- Resources that charge at zero usage (pay-for-availability) are easily forgotten recurring charges: do not deploy them by default. If necessary, use the §2.2 exception process and state their monthly cost.
- Every deployment/change report must include the **`本次新增计费点` list** (write `无` if none), so the user always knows what makes up the bill.
- Record approved billing points in **`## 计费点清单` in the project-root `AGENTS.md`** (§2.1): service, billing model, expected monthly cost, approval date, quota (scope / current-period allowance / overage behavior) and sharers (§17.3). Approval remains effective and inspectable across sessions: prevent repeated questions and, more critically, treating an unapproved item as approved.

### 1.2 Automated paid-service chains must have hard limits
Before deploying or running any chain where code / timers / events / scripts trigger paid calls (including local test scripts), check every item below. This section bounds **call count**; §17 gates each operation's **volume**:
1. **No self-triggering loop:** map the event flow. Can a handler's outputs (storage writes / messages / logs) directly or indirectly trigger itself again? A loop prohibits deployment. If a loop is necessary (bounded retry orchestration or recursive task processing), use the §2.2 exception process and impose an **explicit recursion-depth / iteration limit**.
2. **Bound retries:** explicitly set maximum retries, backoff and a dead-letter/abandonment path. **Never rely on platform-default retries**, which are often unlimited or excessive.
3. **Cap concurrency and frequency:** explicitly limit maximum concurrency, call frequency and batch size. Use the longest schedule interval the business can tolerate.
4. **Timeouts and circuit breaking:** give every outbound call a timeout; automatically stop on an abnormal spike in error rate or call volume (circuit breaker / alert-triggered disablement).
5. **Predefine a one-switch stop:** before starting, identify the switch to stop runaway activity (disable the trigger / set concurrency to 0 / revoke the key). Write it in deployment instructions; do not invent it during an incident.

### 1.3 Put billing defenses in place before deployment
- Before or with the first deployment, establish at least two budget alerts: **2× expected monthly cost** (anomaly warning) and a **runaway-level amount** (immediate action). Verify the notification channel actually reaches the user; mail sent to an unread inbox is no alert.
- During the first billing cycle after launch, proactively compare actual and expected charges once and explain any deviation to the user.
- On abnormal spend, **stop the source first, then investigate the cause**, never the reverse. Report immediately; the user decides appeals, cleanup and other remediation.
- Hard-stop quotas (free daily/monthly allowances) do not charge for overage, so budget alerts miss exhaustion. If supported, configure usage alerts at **50% / 80% of the current-period allowance**. Otherwise mark `无告警` in `## 计费点清单` (§17.3).

### 1.4 Self-check keywords: immediately recheck this section when writing/seeing them
- "Within the free allowance; activate it first / no need to tell the user."
- "The quickstart includes this service by default; enable it while here."
- "The platform retries failures automatically; no need to manage it."
- "It is fine for the worker to trigger itself / the event loop will not actually run."
- "Configure billing alerts later."
- "Loop the test-script calls; running all night is fine."

## 17. Estimate metered operations first: a per-operation volume gate, including read-only, manual, and one-off operations

### 17.1 Core principles
- **A metered operation** is any operation whose cost grows with the amount of data scanned, traversed, or transferred and that hits a **quota-limited or usage-billed backend**: cloud database queries (rows read/written), object-storage listing/export, log and analytics queries (bytes scanned), retrieval/vector-index scans, outbound traffic, and batch model calls. “Quota” means a daily/monthly allowance accumulated within a window whose exhaustion stops service or incurs charges, not an instantaneous rate limit. **If unsure whether a backend has a quota, treat it as having one.** **Reads also consume quota**; reads and writes spend the same kind of money against the quota.
- **Cost depends on volume, not count.** §1.2 governs call counts—loops, retries, concurrency. This section governs per-operation volume: one nested full-table COUNT query can read 160 million rows in a single execution, burning 33 times the free daily allowance. Both dimensions must pass their gates. “Count = 1” exempts only §1.2, not this section.
- **Where code runs is not where its cost lands**: a local tool with `--remote`, a cloud endpoint, or production credentials performs a cloud metered operation. **A statement/command supplied directly by the user is not exempt**: the user specified what to query, not approval to scan N rows.
- **Quota exhaustion cannot be rolled back**: a read cannot be reverted. Only the quota-window reset restores it; until then, **all callers** of the pool, including production traffic from other projects in the same account, receive errors. The gate must therefore precede execution; there is no after-the-fact remedy.

### 17.2 Mandatory actions before every metered operation; none may be skipped
1. **Identify the pool**: first obtain the effective account ID using §7.4 step 2. For account-scoped pools, identity verification on reads is no longer merely recommended here. Then consult `## 计费点清单` (§§1.1 / 17.3): which pool is consumed, its current-window allowance, whether overage hard-stops service or becomes billed usage, and every sharer. If these fields are missing, first fill them from platform documentation and the account resource inventory (§2.1; ask the user if unavailable), then execute.
2. **Estimate at zero cost**: obtain a **number** using `EXPLAIN` / `EXPLAIN QUERY PLAN`, dry-run, or metadata, in the quota's units—rows, bytes, objects, or requests. For unindexed `SCAN`, `EXISTS` / correlated subqueries, or unindexed JOINs, estimate by **multiplying row counts across layers**, not adding them. `SEARCH USING INDEX` proves index use, not selectivity; for a non-primary/nonunique index, use the number of rows matching that value. Obtain each layer's row count from zero-cost sources such as `sqlite_stat1`, `MAX(rowid)`, console metrics, or an inventoried size; when uncertain, use the full-table row count as an upper bound. An unfiltered `COUNT(*)` reads the whole table itself and must not be used as an estimator. For BigQuery/Athena-like systems, dry-run the bytes scanned. For object storage, limit by prefix and probe one page with `--max-items`, then extrapolate. For logs, require a time window and limit; first run a 1-hour window, then extrapolate. `LIMIT` limits **output** rows, not **scanned** rows: COUNT, aggregation, and EXISTS still scan the full table.
3. **No numeric estimate → first reduce to a bounded probe**, run it once, and extrapolate. The predicate must use an index, **then** add LIMIT / time window / prefix / sampling; LIMIT alone is not a bound. The probe itself must first pass steps 2 and 4. If a number is still unavailable, **do not execute**; use the 🛑 section (§9.6.1).
4. **Compare thresholds**: estimated volume **> 1% of that pool's current allowance**, or this session's cumulative volume against the same pool **> 10%** → **do not execute**; use the 🛑 section.
   “Current allowance” means the platform's current-window allowance for a hard-quota pool; for a purely usage-billed pool, it means the approved expected monthly volume in the inventory (expected monthly cost ÷ unit price). If missing, return to step 1 and fill it. When usage consumed can be obtained, compare against the **remaining allowance**. If the pool includes a production project, halve the thresholds to 0.5% / 5%.
   The 🛑 choices are fixed: A, add an index / rewrite the query to use an index, then re-estimate (recommended in development when schema changes are permitted: zero quota); B, export a local copy before querying (recommended for multiple diagnostic queries: pay once; estimate the export itself under this section, and if over threshold phrase the choice as “批准导出占 X%” without nested approval; delete the copy after use); C, the user approves this operation's overage, stating its percentage of the allowance and effects on sharers, including production service suspension if the pool contains a production project. The default is no execution. After the answer: A, re-estimate and ask again if still over; B, finish all diagnostics locally after exporting, then report; C, execute and perform step 5. Below the thresholds, execute directly without pausing to ask (§9.2, small choices).
5. **Check after execution**: put provider-reported actual usage, such as `rows_read` / bytes scanned, and its percentage of the current allowance in the final report's 💰 section (§9.7). Explain deviations **> 10×** from the estimate: an incorrect estimation method may be wrong by even more next time.
- **From the second diagnostic/exploratory query in the same session against the same cloud backend onward**, choose one: export a local copy under step 4B and run locally, or pass steps 1–4 separately for every query. Doing neither is forbidden.
- **Metered queries embedded in code or scheduled tasks**: before deployment, use step 2 to estimate per-operation volume × the §1.2 frequency limit = daily volume. Daily volume **> 10% of the daily allowance** goes to the 🛑 section under step 4. Record the estimate in that query's source comments.

### 17.3 Structure: shared quota is coupling and must be declared; production does not share by default
- In `## 计费点清单` (§§1.1 / 2.1), add one row per metering dimension—for example, rows read/day, rows written/day, storage—with **quota scope** (account/resource), **current allowance**, **overage behavior** (hard stop/billing), and **sharers** (other projects using the same pool). Writing “无” (none) requires checking every resource under the account; never assume it. Account-scoped quota is particularly easy to miss: Cloudflare D1's free allowance, for example, is per account, not per database.
- **A production project's (§0.1) account-scoped hard-stop quota must not be shared with any other project by default**. Move it to a separate account or upgrade to a tier that bills overage; a new billing point still needs §1.1 approval. Continued sharing is a §2.2 exception: record it in that production project's `## 例外清单` with a remediation plan and date. Every project in the pool then uses step 4's halved thresholds. Remediate in §0.3's small steps, never through an unprotected change.
- **§7.3 isolates credentials; this section isolates quota**. Project-specific profiles pointing to the same account can satisfy §7.3 while failing this section.

### 17.4 Self-check keywords: immediately reread this section when writing or seeing them
- “read-only, no data changes, so safe” / “only once, not a loop, so §1.2 does not apply”;
- “a local tool, just querying with `--remote`” / “free tier, so it costs nothing” / “the SQL came from the user”—none is an exemption;
- “cannot estimate; run first and stop if slow”—the backend has already scanned and spent quota that stopping cannot recover; “the rule says export, so export directly”—the export itself must first be estimated;
- “it has LIMIT” / “the table is small, so SCAN is fine” without multiplication / “EXPLAIN says SEARCH, not SCAN” without checking index selectivity;
- “this is a dev database, not production” / “only I use this account”—quota is shared within an account, and the sharers have not been checked;
- “the same query was fine last time” / “the user needs results urgently, so estimation is too cumbersome”;
- “the inventory is missing fields; fill them after running” / “loosen thresholds for a dev project”—thresholds belong to the pool, not the project's phase.

### 17.5 Relationship to existing safeguards
- Independent of **§1.2**: §1.2 limits count—loops, retries, concurrency—while this section limits per-operation volume. Multiply both for queries in code (the final item of 16.2).
- **§§1.1 / 1.3**: approval of a billing point authorizes using the resource, not unrestricted scanning in any operation. This section adds inventory fields to §1.1. Exhausting hard-stop quota incurs no charge and escapes budget alerts; §1.3 therefore requires separate usage alerts.
- Independent of, and dependent on, **§7**: §7 governs whose identity is used; this section governs how much is spent. §7.1's recommendation for reads concerns identity verification. Step 1 here requires an account ID to identify the pool, so reads must still perform §7.4 step 2. Credential isolation under §7.3 is not quota isolation under §17.3.
- The same basis as **§10**: quota exhaustion is irreversible—it cannot be reverted, only reset when the window ends—so the gate precedes execution.
- **§§9.2 / 9.6.1**: an above-threshold stop is a rule gate requiring prior approval, case 2, and uses the 🛑 format. Do not pause below the threshold.
- **§13.4**: one-off diagnostic SQL is not a deliverable and does not use the acceptance checklist, but must still pass this section's gates.
