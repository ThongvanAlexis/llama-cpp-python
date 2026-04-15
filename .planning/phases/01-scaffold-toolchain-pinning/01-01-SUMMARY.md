---
phase: 01-scaffold-toolchain-pinning
plan: 01
subsystem: infra
tags: [github-actions, ci, workflow, actionlint, ban-grep, windows, cuda]

# Dependency graph
requires:
  - phase: 01-scaffold-toolchain-pinning
    provides: "Phase 1 research, context, validation, and plan documents (no code prerequisites — Wave 0 substrate originates here)"
provides:
  - ".github/workflows/build-wheels-cuda-windows.yaml — two-job skeleton (preflight stub + operational lint-workflow) with workflow_dispatch trigger and 5 inputs"
  - "Operational ban-grep self-check (TC-10) scoped to this file only — exactly 1 occurrence of the banned literal (the grep pattern itself); paraphrases used everywhere else"
  - "actionlint step on Ubuntu lint-workflow job (Claude-discretion extension per RESEARCH.md OQ4) — installed via official one-line curl script"
  - "Strict-mode shell prelude pattern (`set -euo pipefail` for bash) for Plan 02/03 reuse"
  - "Job layout contract: parallel preflight (windows-2022) + lint-workflow (ubuntu-latest), no needs: dependency"
  - "Dispatch input contract: python_version/cuda_version (choice), msvc_toolset (string), verbose/dry_run (boolean) consumable via ${{ inputs.* }} in Plan 02"
affects: [01-02-toolchain-install, 01-03-forensics-doc04, phase-2-build-cache]

# Tech tracking
tech-stack:
  added:
    - "actionlint v1.7.12 (workflow static analysis — install via official curl one-liner, no GitHub Action wrapper)"
    - "actions/checkout@v4 (lint-workflow only; submodules not requested — this job only reads the workflow file itself)"
  patterns:
    - "Strict bash prelude: `set -euo pipefail` at top of every multi-line `shell: bash` run block"
    - "Step-name paraphrasing for ban-grep — all diagnostic strings use 'banned CMake flag' / 'compiler-bypass workaround' instead of the literal flag name, so `grep -c -F` returns exactly 1 (the grep command itself)"
    - "Two-job parallel layout: cheap Ubuntu lint runs alongside expensive Windows preflight with no needs: edge — fast feedback on workflow-syntax regressions without paying Windows runner minutes"
    - "PLACEHOLDER step inside preflight stub (Write-Host one-liner) so the YAML validates; marked with comment for Plan 02 to delete cleanly"

key-files:
  created:
    - ".github/workflows/build-wheels-cuda-windows.yaml — Phase 1 scaffold (77 lines, two jobs, fully operational lint-workflow + preflight stub)"
  modified: []

key-decisions:
  - "Bash strict prelude `set -euo pipefail` chosen as the Plan 02/03 reusable idiom for all `shell: bash` blocks (vs naked `set -e` only) — fail loud on unset vars and pipe failures, not just on individual command exit"
  - "actionlint installed via official `download-actionlint.bash` curl one-liner (NOT via a wrapping GitHub Action like rhysd/actionlint-action) — matches the 'no external dependencies beyond what the job needs' ethos and avoids action-pin surface area; locked decision for Plan 03"
  - "Ban-grep diagnostic messages use paraphrases ('banned CMake flag', 'compiler-bypass workaround flag') instead of the literal flag name — preserves the count-equals-1 invariant from VALIDATION.md so the `| grep -q '^1$'` check in the quick command stays meaningful"
  - "PLACEHOLDER step inside preflight stub uses minimal `Write-Host` one-liner (vs an empty `steps: []` which actionlint rejects) — Plan 02 deletes this cleanly, marked with a comment"
  - "actions/checkout@v4 in lint-workflow does NOT use `submodules: recursive` — the lint job only reads the workflow file itself; the submodule pull is reserved for the preflight job in Plan 02"

patterns-established:
  - "Pattern 1: Two parallel jobs (Ubuntu lint + Windows preflight) with no `needs:` edge — Ubuntu re-runs ban-grep + actionlint on every dispatch, catching regressions without re-paying Windows runner minutes"
  - "Pattern 2: Ban-grep with paraphrased diagnostics — the literal banned flag appears EXACTLY ONCE in the file (the grep pattern itself); all step names, error messages, and OK messages use paraphrases"
  - "Pattern 3: actionlint via official curl install (not a wrapping action) — keeps action-pin surface minimal"
  - "Pattern 4: workflow_dispatch-only trigger + 5 typed inputs (choice/string/boolean) — no auto-triggers (push/PR/release/schedule) per WF-02; choice inputs render as dropdowns in the GitHub UI for self-documenting valid sets"

requirements-completed: [WF-01, WF-02, WF-03, WF-04, TC-01, TC-10]

# Metrics
duration: 2 min
completed: 2026-04-15
---

# Phase 1 Plan 1: Scaffold & Lint Job Wiring Summary

**Two-job Windows CUDA wheel workflow scaffold with operational ban-grep + actionlint on parallel Ubuntu lint job — preflight job stubbed for Plan 02, all 5 dispatch inputs and TC-10 invariant in place.**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-15T20:55:59Z
- **Completed:** 2026-04-15T20:57:31Z
- **Tasks:** 1
- **Files created:** 1 (`.github/workflows/build-wheels-cuda-windows.yaml`)
- **Files modified:** 0

## Accomplishments

- Created `.github/workflows/build-wheels-cuda-windows.yaml` (77 lines) — new file, not an edit of upstream's `build-wheels-cuda.yaml` (WF-01 merge-conflict hygiene preserved; `git diff` against upstream file returns empty).
- `on: workflow_dispatch` is the ONLY trigger (WF-02) — no `push`, `pull_request`, `release`, `schedule`, or `workflow_call`. All five dispatch inputs present with exact defaults and types specified in the plan's `<interfaces>` block (WF-03, WF-04).
- `preflight` job stub: `runs-on: windows-2022` literal (TC-01 — never `windows-latest`), `timeout-minutes: 20`, `defaults.run.shell: pwsh`, single placeholder step that Plan 02 will delete.
- `lint-workflow` job FULLY OPERATIONAL: `runs-on: ubuntu-latest`, `timeout-minutes: 5`, no `needs:` (parallel with preflight), 3 steps — `actions/checkout@v4`, ban-grep (TC-10) scoped to this file only with paraphrased diagnostics, actionlint via official curl one-liner.
- Ban-grep invariant achieved: `grep -c -F 'allow-unsupported-compiler'` returns exactly 1 (the grep pattern literal at line 66) — every other diagnostic uses paraphrases ('banned CMake flag', 'compiler-bypass workaround flag').
- `permissions: contents: read` at workflow level (least privilege; Phase 4 publish job will widen via job-scoped override per RESEARCH.md Pitfall 7).
- Local validation: `actionlint v1.7.12` (downloaded via official one-liner) exits 0 against the new file; quick command from VALIDATION.md (`actionlint ... && grep -c -F ... | grep -q '^1$'`) returns PASS.

## Task Commits

Each task was committed atomically:

1. **Task 1: Create workflow skeleton with trigger, inputs, preflight stub, and operational lint-workflow job** — `dfe478b` (feat)

**Plan metadata:** *(pending — committed after this SUMMARY.md is written)*

## Files Created/Modified

- `.github/workflows/build-wheels-cuda-windows.yaml` (CREATED, 77 lines) — Phase 1 scaffold. Top-of-file placeholder preamble (Plan 03 replaces with full DOC-04), `name: Build Wheels (CUDA Windows)`, `on: workflow_dispatch` with 5 inputs, `permissions: contents: read`, two jobs (`preflight` stub + operational `lint-workflow`).

## Decisions Made

- **Strict bash prelude `set -euo pipefail`** chosen for all `shell: bash` blocks — fails loud on unset vars and pipe failures, not just on individual command exit. Plan 02/03 should reuse this exact prelude in any bash steps they add.
- **actionlint via official `download-actionlint.bash` curl one-liner** — NOT via a wrapping GitHub Action like `rhysd/actionlint-action`. Matches the "no external dependencies beyond what the job needs" ethos and avoids action-pin surface area. Locked decision for Plan 03 — do not re-decide.
- **Ban-grep diagnostic messages use paraphrases** ('banned CMake flag', 'compiler-bypass workaround flag') instead of the literal flag name — preserves the count-equals-1 invariant from VALIDATION.md so the `| grep -q '^1$'` quick-command check stays meaningful. This is the precedent for Plan 03 Task 2's DOC-04 inline comments.
- **PLACEHOLDER step inside preflight stub** uses minimal `Write-Host` one-liner (vs an empty `steps: []` which actionlint rejects). Plan 02 should delete this step cleanly when populating the preflight body — it is marked with `# PLACEHOLDER — delete when Plan 02 populates preflight body`.
- **actions/checkout@v4 in lint-workflow has NO `submodules:` clause** — the lint job only reads the workflow file itself. Submodule pull is the preflight job's concern in Plan 02.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Initial draft contained the banned literal in 4 places (step name, grep pattern, error message, OK message), violating the count-equals-1 invariant**
- **Found during:** Task 1 (initial Write of workflow file before commit)
- **Issue:** First draft of the ban-grep step used the literal flag name `--allow-unsupported-compiler` in the step `name:`, in the `::error` message, and in the success `echo "OK: ..."` message in addition to the grep pattern itself. `grep -c -F` returned 4, but VALIDATION.md and the plan both require exactly 1 (the grep pattern literal).
- **Fix:** Rewrote step name to `'Lint: ban the compiler-bypass workaround flag'`, error message to `the banned CMake flag is present (upstream #1543)`, and OK message to `OK: no banned compiler-bypass workaround flag in $WORKFLOW`. Grep pattern itself unchanged — that's the one legitimate occurrence.
- **Files modified:** `.github/workflows/build-wheels-cuda-windows.yaml`
- **Verification:** `grep -c -F 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml` returns 1; `grep -nH ...` shows the only line is `if grep -nH 'allow-unsupported-compiler' "$WORKFLOW"; then` (the pattern literal itself).
- **Committed in:** `dfe478b` (caught and fixed pre-commit, so single atomic Task 1 commit contains the correct version)

---

**Total deviations:** 1 auto-fixed (1 bug — pre-commit catch).
**Impact on plan:** Zero scope creep. The fix preserves the ban-grep invariant exactly as VALIDATION.md and the plan required; without it, every per-task quick-command run from this point forward would have failed `| grep -q '^1$'`. Establishes the precedent for Plan 03 Task 2's DOC-04 comment-text policy: paraphrase the literal everywhere except the single grep pattern.

## Issues Encountered

- Local dev machine has neither `actionlint` nor `yq` installed. Resolved by:
  1. Installing `actionlint v1.7.12` via the official `download-actionlint.bash` one-liner to `/tmp/actionlint.exe` (same install method the lint-workflow job uses on Ubuntu — validates that the install path itself works on Windows-host bash).
  2. Verifying YAML structure with Python's `yaml.safe_load()` (always available on Windows per global CLAUDE.md `python` aliasing) instead of `yq`. All structural assertions from the plan's `<verify>` block confirmed: 5 inputs present, 2 jobs (`lint-workflow`, `preflight`), preflight runs-on `windows-2022` literal, lint-workflow runs-on `ubuntu-latest` with `needs: None`, permissions `contents: read`.
- VALIDATION.md noted this could happen ("If neither is available, the `lint-workflow` job itself will catch issues on the next dispatch — but local feedback is 2s vs 12min so it's worth the one-time setup"). Local `actionlint` install completed in ~3s and reduces feedback latency for Plans 02/03 from 12 min (dispatch) to 2s (local).

## Authentication Gates

None — no external service authentication required for this plan.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- **Ready for Plan 02 (toolchain install + preflight assertions):** The substrate is in place. Plan 02 deletes the placeholder `Preflight stub` step and populates the preflight body with submodule checkout, ilammy/msvc-dev-cmd activation, mamba CUDA install, LongPathsEnabled, and the 6 assertion steps (assert-msvc, assert-nvcc, assert-cuda-single-path, assert-submodule, assert-python, discover-vs).
- **Ready for Plan 03 (forensics + DOC-04):** Top-of-file preamble is a placeholder; Plan 03 replaces it with the full DOC-04 documentation block. Inline comments at the 4 DOC-04 sites (msvc_toolset input, ilammy step, lint-workflow job, MSVC probe step) are Plan 03's concern. Critical: Plan 03 Task 2 must use paraphrases in all comment text — the precedent established here (count-equals-1 invariant) is the policy.
- **No blockers.** Pre-plan risks from STATE.md (MSVC 14.39 component availability, hashFiles submodule indirection) remain pre-plan risks for Plan 02; nothing in this plan resolves or aggravates them.

---

## Self-Check: PASSED

Verifications performed after writing this SUMMARY.md:

- `[ -f .github/workflows/build-wheels-cuda-windows.yaml ]` → FOUND (created in Task 1).
- `git log --oneline --all | grep -q "dfe478b"` → FOUND (Task 1 commit `dfe478b` exists).
- `grep -c -F 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml` → returns `1` (ban-grep invariant preserved).
- `actionlint .github/workflows/build-wheels-cuda-windows.yaml` → exits 0.
- `git diff .github/workflows/build-wheels-cuda.yaml` → empty (upstream file untouched per WF-01).

All claims in this SUMMARY verifiable against disk and git history.

---
*Phase: 01-scaffold-toolchain-pinning*
*Completed: 2026-04-15*
