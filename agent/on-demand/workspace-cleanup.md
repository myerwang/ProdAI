---
topic: agent-governance
applies_to: [ai-agents, git, development, production]
data_scale: [small, medium, large]
decision: read this domain file in full when the resident entry trigger matches
part: on-demand/workspace-cleanup
source: myerwang/ProdAI agent/AGENTS.md @ tag agents-baseline-20261028 (commit f301b766)
language: en (operational)
status: stable
last_reviewed: 2026-10-28
---

# Build artifacts and task temporary files

**Read when:** Read after every successful build, before producing task temporary files outside the repository (recordings, probe output, packages, source copies, downloads), before any cleanup, and before the final report when such files exist.
**Covers source sections:** §15.
**Text:** the verified full English translation of the source sections, wording unchanged; only blank-line runs and `---` separators were compacted. Section numbers (§) are the source numbers and are cited by the entry and the other files.
**Entry:** `../AGENTS.md` (always loaded; this file is read on the triggers above).

## 15. Build artifacts and task temporary files: clean after use; clean only what you created

### 15.1 Build artifacts
- **After every successful build, inspect and clean temporary files and obsolete build outputs confirmed to be useless and regenerable**. A full cache purge is not required each time.
- Keep currently valid outputs, useful incremental caches, release/signing packages, and material still used by active builds. Material of unknown ownership, non-regenerable material, or material whose rebuildability is uncertain must be preserved and reported.
- Use cleanup methods supported by the project's technology stack and toolchain. Never delete cache-internal files solely by modification time (mtime).
- When retention depends on generation counts or space thresholds, follow existing project configuration. Without such configuration, do not assume a version or output is useless: report candidates and their space usage, then await an explicit retention strategy.

### 15.2 Task temporary files, including files outside the repository
Files I create for a task outside the repository also require cleanup: temporary directories under `/tmp` or `$TMPDIR`; performance recordings (Instruments `.trace` / `.ktrace`, profiler output); probe/regression/load-test output directories; simulator or packaging intermediates (`.app`, `.ipa`, unpacked directories); source copies made for isolated builds; and downloaded, extracted archives. One such file can occupy hundreds of MB to several GB. Repeatedly creating a new directory for each probe can consume tens of GB within days.

- **Choose the location before generating files**: put one task's temporary files under one identifiable root. Prefer the tool's session-specific scratchpad; otherwise use `mktemp -d` with project and task prefixes in the directory name. Do not scatter new directories directly under `/tmp`.
- **Reuse directories for the same kind of probe**: repeated probes/recordings overwrite the same output location or retain only the latest result. Do not accumulate a new directory per run. When comparing runs, extract conclusions or metrics before deleting raw recordings.
- **Delete as soon as conclusions are extracted**: immediately remove large intermediates—recordings, packages, unpacked directories, source copies—after extracting conclusions or metrics, or completing the upload. In scripts, use `trap` to clean the temporary directories the script created on exit, including failure.
- **Mandatory final check**: before the final report, inspect the temporary files this task created. For anything still needed—for user comparison or a later step—report its path, size, and deletion condition. Do not silently leave it behind.
- **Clean only what you created**: delete only paths provably created by this task/session, identified by recorded paths or dedicated prefixes. Do not delete others' files based on similar directory names, modification times, or an appearance of being temporary. Even large temporary files from other projects or processes must be left untouched; report only their source and size (§10.1).

### 15.3 Boundaries
- This section grants no authority to expand deletion scope, permanently delete material, or add background cleanup, build hooks, or automatic cleanup mechanisms. Existing approval, deletion, and recovery safeguards remain applicable. Cleanup must never touch unconfirmed targets.
- Anti-patterns—stop immediately when writing them: “the system will clean the temporary directory”; “keep it in case it is useful later” without a path or deletion condition; “create a new directory for every probe to distinguish them”; “recording the trace finishes the task; the files need no attention.”
