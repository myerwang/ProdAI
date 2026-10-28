---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/deploy-and-rollback
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# Deployment, account verification and production rollback

**Read when:** Read before any deploy / release / upload / publish, before any account-sensitive write (cloud CRUD, signing, profile/context switch, first push of a public repo), before any production change (§0.3 recovery plan), and when a GUI step seems necessary (§11). Re-read when such an action first appears mid-task.
**Covers source sections:** §0.3, §7, §11.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

<!-- §0.3 is extracted here from §0; §0.1–0.2 and §0.4–0.5 are in core-fulltext.md and condensed in the entry. -->
### 0.3 Production mode: minimal changes + mandatory recovery plan
- **Completely overturning the existing architecture is not an option.** Architecture evolution must be a gradual replacement: independently verifiable small steps that can be stopped and rolled back whenever a problem appears.
- **Change one thing at a time:** make each change small enough that a failure is immediately attributable to it. Do not bundle unrelated changes "while here."
- **Before every production change, prepare the recovery plan.** All four elements are mandatory:
  1. **Code anchor (Git, mandatory):** create an annotated tag on **the commit currently running in production** and push it immediately:
     `git tag -a anchor/<yyyymmdd-HHMM>-<变更slug> -m "<变更摘要 + 回滚命令>"` → `git push origin <tag名>`.
     Recover by redeploying the anchor tag or rolling forward with `git revert`; **never erase history with force push**.
  2. **Data anchor:** snapshot/back up before touching databases or storage. Use expand-contract for schema changes: add only in this change; delete columns/tables in a later, independent change. Never perform irreversible deletion, overwrite or migration without a backup.
  3. **Configuration/infrastructure anchor:** retain the previous directly redeployable IaC/configuration state (previous template, stack version or configuration export).
  4. **Written rollback plan:** specify rollback triggers, exact commands and expected recovery time. Steps must be executable now, not invented when failure occurs. Put rollback commands in the anchor tag message (already included in item 1), the full plan in this change's PR/commit description, and the full plan additionally in the project-root `AGENTS.md` for complex changes affecting architecture.
- Validate after the change and observe it for a window; proceed to the next change only after confirming stability.
- When production architecture violates constraints (§1, §4, etc.), schedule remediation immediately, but follow this section's small steps; do not make unprotected changes or overturn the architecture.

## 7. Account verification: a prerequisite for account-sensitive operations

### 7.1 Core principles
- **Before any account-sensitive operation**, explicitly verify that the **currently effective account/identity equals the account declared by the project**, and act only after verification passes. This includes remote deployment; production changes; cloud-resource CRUD; **app signing, packaging and distribution** (developer Team / App Store Connect / Google Play Console / certificates and provisioning profiles); configuration, credential, profile or context changes; and APIs authenticated or billed by account.
- **A write to the wrong account harms two projects at once**: the intended project never receives its change, causing a missing deployment or inconsistent state, while the other project receives or has resources overwritten with material that does not belong there, causing contamination or destruction. Apply every requirement in this section as protection against an accident that will happen, not one that “probably will not”.
- **Subaccounts are distinct accounts too**. Member accounts, subaccounts and assume-role targets under the same organization/root are different accounts. The right organization with the wrong member account is still a two-project accident. Verify **unique IDs**—account ID / subaccount ID / role ARN / Team ID / project ID—not aliases, display names or visual resemblance.
- **Accounts are not limited to cloud accounts**. They include infrastructure accounts (AWS/GCP), app developer/distribution accounts (Apple Developer Team, Google Play), code hosting (GitHub), package/image registries (npm/registry), and Git author identity. Anything authorized by identity where choosing incorrectly can cause harm is covered.
- **An activation script printing “switched to account X” does not prove that X is effective**. Use **actual identity-query results**, such as AWS `aws sts get-caller-identity`, GCP `gcloud config list account`, Kubernetes `kubectl config current-context`, GitHub `gh api user`, npm `npm whoami`, and iOS signing `security find-identity -p codesigning -v` together with project `DEVELOPMENT_TEAM` / `ExportOptions teamID`. Do not rely on activation-script echoes or assumptions.
- Verification before **writes**—deployment, deletion, configuration changes and push—is **mandatory**, not optional. For reads it is recommended on the identity dimension; the **read-cost** gates in §17 are mandatory.

### 7.2 Projects must declare account bindings in a fixed location
- Every project must declare its bound accounts **at unique-ID granularity**: cloud account/subaccount ID and region; project-specific profile/context name; app developer Team ID, Bundle ID, signing certificate/provisioning profile and distribution-channel account; Git author; deployment target/resource ID; image/package registry, and similar identities. The verification baseline must live in **`## 账号绑定` in the project-root `AGENTS.md`** (§2.1). Large resource inventories may be split into a separate account inventory linked from `AGENTS.md`.
- **If the project has no declared account binding**, stop before an account-sensitive operation and ask the user which account to use. Act only after an explicit answer has been recorded in the project’s fixed configuration location. **Never guess from credentials that happen to be present or a global/default profile.**

### 7.3 Physically isolate credentials by project to prevent account mix-ups
- **One project, one specially named profile/context/configuration containing the project name**. Every cloud command for that project must explicitly select it, for example `--profile <项目>`, command-scoped `AWS_PROFILE=<项目>`, `kubectl --context`, or `gcloud --configuration`. Do not depend on whatever the current shell happens to have activated.
- **Never use a default profile or global environment-variable credentials for an account-sensitive write**. Credentials that merely happen to be in the environment are precisely the channel through which project A writes into project B’s account.
- **Switching projects means starting with a clean environment**. Before acting, inspect and clear the preceding project’s leftover `AWS_*` / `GOOGLE_*` / `KUBECONFIG` / `GH_TOKEN` and similar variables. Reusing an old shell to deploy a new project is high risk.
- **Prefix deployment-unit names with the project name**—stacks, services and resource groups. Even in the wrong account, an unfamiliar extra stack is preferable to overwriting another project’s same-named resource. A **same-named resource belonging to another project** in the target account is a strong wrong-account signal: stop immediately and verify.

### 7.4 Mandatory steps before an account-sensitive write; none may be skipped
1. Read the project’s fixed configuration location and obtain the **expected account’s unique ID**. If absent, ask the user under §7.2.
2. Run the identity query with the project-specific profile/context and obtain the **currently effective account’s unique ID**.
3. **Compare the IDs character for character**. If they differ, **stop immediately**, report “expected X, actual Y”, correct activation/credentials, and **verify again**. Never continue with a mismatch.
4. Only after a match may the write proceed. A **second verification of the result** is recommended afterward, such as fetching the online configuration and diffing it against local configuration.

### 7.5 Self-check keywords and situations: immediately perform §7.4 when any appears
- “Deploy for me / push to production / deploy / update / apply / push to the cloud / go live / publish to an app store”.
- “build / archive / signing / export ipa / upload to TestFlight·App Store Connect·Play Console / altool / notarytool / fastlane / xcrun”: signing Team / Bundle ID / provisioning profile / certificate must equal the project’s declaration.
- “source xxx/activate / switch profile / change context / export AWS_/GCP_/KUBE_/GH_TOKEN”.
- Running an activation script through a **pipe or subshell**, such as `source ... | ...` or `(source ...; ...)`, then running an account-sensitive command: activation very likely **did not reach the current shell**.
- “Default credentials should be fine / there is a key in the environment, use it / use global ~/.aws”.
- **Switching subaccounts or assuming a role within one organization, or sharing one machine/shell across projects**: common account-mix-up situations.
- Creating a public repository / first commit / first push: verify **both account and author**, together with §3.

### 7.6 Response after a wrong-account operation has occurred
1. **Stop all further operations immediately and preserve the scene**. Do not make another opportunistic change to hide or repair the mistake.
2. **Inventory both sides’ impacts**: what is missing from the intended account—the changes that never arrived—and what was added or changed in the mistaken account. List each item.
3. **Report to the user**: expected X, actual Y, the impact inventory for both sides, and possible remedies such as rollback/deletion on the mistaken side and deployment to the intended side.
4. **Wait for the user’s decision before remediation**. Verify identity under this section before cleaning the mistaken account too; do not make a second account mix-up in the rush.
5. Review and fix the channel that caused the mix-up—leftover environment, default profile or undeclared binding—and add project rules if necessary.

### 7.7 Relationship to the existing safeguards
- Combine this with **§3 (personal information)**: §3 prevents unauthorized use of the user’s identity information; this section prevents operations in the wrong account. Public-repository and deployment operations must satisfy both.
- This is a **prerequisite gate**, independent of §1 (cost safety) and §4 (managed-first selection). Cloud-resource operations must pass all three.

## 11. Publishing, deployment and upload: prefer CLI/API; GUI requires justification

### 11.1 Core principle
- **When an official or mature CLI/API can publish, deploy or upload, command-line execution must be preferred**: official CLI, official API, or a mature toolchain built on them. **GUI is allowed only when command-line execution is actually unavailable.**
- Reason: CLI/API operations are reproducible, auditable, scriptable and recordable in change history, so failures can be replayed precisely. GUI clicks cannot be reconciled afterward or connected to §7 account-verification commands (identity query → exact comparison).

### 11.2 When command-line execution is actually unavailable (at least one condition must hold)
- No official CLI/API exists for the operation, or official documentation explicitly requires this step to be completed in the console/web administration interface.
- A CLI/API exists but **does not work in the current scenario** (for example, first-time activation requires one-time GUI authorization or acceptance of terms), and this has been **actually tried or verified**. "I do not know the command" / "a few GUI clicks are faster" does not establish unavailability.
- Once the exception is established, explain to the user **which step** and **why the CLI is unavailable** before using GUI. Afterward, record the actual GUI actions in the change description (PR / commit description) to preserve traceability.

### 11.3 Self-check keywords: immediately re-examine this section on writing or seeing them
- "Opening the console / uploading through the web admin is faster."
- "A few GUI clicks will do; no need to look up commands."
- "I usually do this in the web UI / the interface is more intuitive."
- "The CLI has too many parameters; web upload is easier."

### 11.4 Relationship to existing safeguards
- **Applies together with §7 account verification**: this section governs the execution interface; §7 governs the identity. CLI use enables the identity-verification commands in §7.4; CLI operations must still pass the §7 gate first.
- **Connects to §0.3 production recovery plans**: CLI operations let rollback commands be recorded individually in the anchor tag and recovery plan. If GUI steps cannot be recorded in the plan, the plan is incomplete.
