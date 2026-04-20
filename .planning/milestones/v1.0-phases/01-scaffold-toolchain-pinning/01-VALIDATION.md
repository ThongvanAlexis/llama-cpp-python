---
phase: 1
slug: scaffold-toolchain-pinning
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-15
refreshed: 2026-04-15
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | actionlint (workflow static analysis) + POSIX `grep -F` (ban-grep, both local and in-workflow `lint-workflow` job) + dispatched GitHub Actions run (`build-wheels-cuda-windows.yaml` preflight-only) |
| **Config file** | `.github/workflows/build-wheels-cuda-windows.yaml` (self-contained; `lint-workflow` job IS the on-runner reproducible test harness) |
| **Quick run command** | `actionlint .github/workflows/build-wheels-cuda-windows.yaml && grep -c -F 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml \| grep -q '^1$'` |
| **Full suite command** | `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.4.1 -f msvc_toolset=14.39 && gh run watch --exit-status` (watches latest preflight dispatch to green on BOTH `preflight` and `lint-workflow`) |
| **Estimated runtime** | Quick: ~2s (actionlint + ban-grep). Full: ~8–12 min (Windows runner cold-start + mamba CUDA install + preflight assertions; no compile in this phase) |

**Ban-grep invariant:** The literal string `allow-unsupported-compiler` appears EXACTLY ONCE in the final workflow file — inside Plan 01's `Lint: ban --allow-unsupported-compiler` step command literal. All DOC-04 comment-text references added by Plan 03 Task 2 use paraphrases (e.g. `the banned CMake flag`, `the compiler-bypass workaround`). The quick command's `grep -q '^1$'` enforces count-equals-1 (not just count-≥-0) so a literal leaking into comments fails fast.

---

## Sampling Rate

- **After every task commit:** Run quick command (`actionlint` + ban-grep count-equals-1). Workflow must parse, and the literal banned flag must appear exactly once (the ban-grep command itself).
- **After every plan wave:** Run quick command. (Full dispatch reserved for wave completion only — it costs ~10 runner minutes.)
- **Before `/gsd:verify-work`:** Full suite must be green — one successful `workflow_dispatch` run reaching the `preflight` job's terminal `Write-Host "✅ Toolchain preflight passed"` marker with green status on BOTH `lint-workflow` and `preflight` jobs. Additionally: one negative-case dispatch (`msvc_toolset=14.01`) confirming hard-fail at the MSVC probe step.
- **Max feedback latency:** Quick = 2s (local). Full = 12 min (dispatch). Acceptable because the `lint-workflow` job re-runs the ban-grep + actionlint checks on every dispatch — catches regressions without manual re-dispatch of the full preflight.

---

## Per-Task Verification Map

*Bound to real plan task IDs after revision on 2026-04-15. Task IDs follow the convention `{phase}-{plan}-T{task}` (e.g. `01-02-T2` = Phase 1, Plan 02, Task 2). Each plan's actual task composition is the authoritative source — this map is a convenience index for feedback-sampling during execution.*

| Task ID | Plan | Wave | Requirements | Test Type | Automated Command | File Exists | Status |
|---------|------|------|--------------|-----------|-------------------|-------------|--------|
| 01-01-T1 | 01 | 1 | WF-01, WF-02, WF-03, WF-04, TC-01, TC-10 | scaffold+lint-job | `actionlint .github/workflows/build-wheels-cuda-windows.yaml && grep -c -F 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml \| grep -q '^1$'` PLUS `yq '.on \| keys'` returns `['workflow_dispatch']`; `yq '.on.workflow_dispatch.inputs \| keys'` lists all 5 inputs; `yq '.jobs \| keys'` returns `[lint-workflow, preflight]`; `yq '.jobs.preflight."runs-on"'` returns `windows-2022` | ✅ W0 substrate | ⬜ pending |
| 01-02-T1 | 02 | 2 | WF-05, TC-02, TC-03, TC-05, TC-09 | toolchain-install | `actionlint ... && grep -c -F 'allow-unsupported-compiler' \| grep -q '^1$'` PLUS `yq '.jobs.preflight.steps \| length'` returns 7; `grep -c -F 'ilammy/msvc-dev-cmd@v1'` = 1; `grep -c -F 'conda-incubator/setup-miniconda@v3.1.0'` = 1; `grep -c -F 'submodules: recursive'` = 1; `grep -c -E 'Jimver/cuda-toolkit'` = 0; `grep -c -F 'LongPathsEnabled'` ≥ 2 | ✅ depends on 01-01-T1 | ⬜ pending |
| 01-02-T2 | 02 | 2 | TC-04, TC-06, TC-07, TC-08 | preflight-assertions | `actionlint ... && grep -c -F 'allow-unsupported-compiler' \| grep -q '^1$'` PLUS `yq '.jobs.preflight.steps \| length'` = 13; `grep -c -E 'id:\s+(assert-msvc\|assert-nvcc\|assert-cuda-single-path\|assert-submodule\|assert-python\|discover-vs)'` = 6; `grep -c -F 'Set-StrictMode -Version Latest'` ≥ 7; `grep -c -F '1900 + '` ≥ 1; `grep -c -E "^\s*1939\s*\$\|['\"]1939['\"]"` = 0; `grep -c -E 'continue-on-error:\s*true'` = 0 | ✅ depends on 01-02-T1 | ⬜ pending |
| 01-02-T3 | 02 | 2 | TC-02+TC-03 (negative-case) | checkpoint:human-verify | `gh run list --workflow=build-wheels-cuda-windows.yaml --limit 2 --json conclusion --jq '.[0].conclusion == "failure" and .[1].conclusion == "success"'` (happy-path green + `msvc_toolset=14.01` red at probe step) | ✅ depends on 01-02-T2 | ⬜ pending |
| 01-03-T1 | 03 | 3 | WF-01 (forensics surface), DOC-04 (transitive) | forensics+remediation | `actionlint ... && grep -c -F 'allow-unsupported-compiler' \| grep -q '^1$'` PLUS `yq '.jobs.preflight.steps \| length'` = 15; `grep -c -F 'if: success()'` ≥ 1; `grep -c -F 'if: failure()'` ≥ 1; `grep -c -F '\$env:GITHUB_STEP_SUMMARY'` ≥ 2; `grep -c -F 'Encoding utf8'` ≥ 2; `grep -c -F 'Toolchain preflight passed'` = 1; `grep -c -E 'steps\.(assert-msvc\|assert-nvcc\|assert-submodule\|assert-python\|discover-vs)\.outputs\.'` ≥ 5 | ✅ depends on 01-02-T2 | ⬜ pending |
| 01-03-T2 | 03 | 3 | DOC-04 | inline-comments+ban-grep-invariant | `actionlint ... && grep -c -F 'allow-unsupported-compiler' \| grep -q '^1$'` (CRITICAL — comments MUST NOT re-introduce the literal) PLUS `grep -c -F '#1543'` ≥ 2; `grep -c -F 'runner-images#9701'` ≥ 1; `grep -c -F '14.40'` ≥ 2; `grep -c -F 'vcvarsall'` ≥ 1; `grep -c -F 'Jimver/cuda-toolkit#382'` ≥ 1; `grep -c -F 'compiler-bypass workaround'` ≥ 3; `grep -c -F 'banned CMake flag'` ≥ 1; `yq '.jobs.preflight.steps \| length'` = 15 (unchanged — comments only) | ✅ depends on 01-03-T1 | ⬜ pending |
| 01-03-T3 | 03 | 3 | Full phase gate (all 16 requirements) | checkpoint:human-verify | `gh run list --workflow=build-wheels-cuda-windows.yaml --limit 2 --json conclusion --jq '.[0].conclusion == "failure" and .[1].conclusion == "success"'` PLUS human-eye review of Summary tab forensics table + DOC-04 comment coverage + `git diff main -- .github/workflows/build-wheels-cuda.yaml` empty | ✅ depends on 01-03-T2 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

### Requirement → Task Coverage

Cross-check that every one of the 16 Phase 1 requirement IDs is bound to at least one row above:

| Requirement | Bound To | Test Type |
|-------------|----------|-----------|
| WF-01 (workflow file exists, parses) | 01-01-T1, 01-03-T1 | scaffold, forensics surface |
| WF-02 (workflow_dispatch only) | 01-01-T1 | trigger-exclusivity (`yq '.on \| keys'`) |
| WF-03 (5 dispatch inputs) | 01-01-T1 | dispatch-inputs (`yq '.on.workflow_dispatch.inputs \| keys'`) |
| WF-04 (no auto-triggers) | 01-01-T1 | trigger-exclusivity (same as WF-02) |
| WF-05 (submodules recursive) | 01-02-T1, 01-02-T2 (assert-submodule) | grep + runtime Test-Path |
| TC-01 (windows-2022 explicit) | 01-01-T1 | `yq '.jobs.preflight."runs-on"'` |
| TC-02 (MSVC 14.39 probe+install) | 01-02-T1, 01-02-T3 (negative case) | grep `ilammy/msvc-dev-cmd` + dispatch |
| TC-03 (ilammy activation) | 01-02-T1, 01-02-T3 | grep + runtime |
| TC-04 (_MSC_VER cap derived from input) | 01-02-T2 | grep `1900 + ` present + `1939` literal absent |
| TC-05 (mamba CUDA single install) | 01-02-T1 | grep `conda-incubator/setup-miniconda@v3.1.0` + `Jimver` absent |
| TC-06 (nvcc matches cuda_version) | 01-02-T2 (assert-nvcc) | runtime regex in assertion step |
| TC-07 (single nvcc path, no CUDA_PATH_V*) | 01-02-T2 (assert-cuda-single-path) | runtime where.exe + env sweep |
| TC-08 (dynamic VS BuildCustomizations) | 01-02-T2 (discover-vs) | runtime vswhere + glob |
| TC-09 (LongPathsEnabled) | 01-02-T1 | grep + runtime Get-ItemProperty |
| TC-10 (ban-grep self-check) | 01-01-T1, 01-03-T2 (invariant preservation) | in-workflow ban-grep step + local quick-command |
| DOC-04 (inline docs at 4 sites) | 01-03-T2, 01-03-T3 | grep `#1543` ≥ 2, `runner-images#9701` ≥ 1, etc. |

Every requirement ID has ≥ 1 task binding. No requirement is unbound.

---

## Wave 0 Requirements

This phase authors a GitHub Actions YAML file — there is no traditional test framework to install. "Wave 0" here = skeleton-scaffold substrate that provides the parse/lint/ban-grep baseline every later task verifies against. **Wave 0 is complete when Plan 01 Task 1 (01-01-T1) commits the skeleton YAML; no separate pre-plan setup is required.**

- [x] `.github/workflows/build-wheels-cuda-windows.yaml` — skeleton with `on: workflow_dispatch` + all 5 inputs and the `preflight` + `lint-workflow` job stubs. Produced by **01-01-T1**. Required before tasks 01-02-T1..T3 and 01-03-T1..T3 can run their automated checks.
- [x] `lint-workflow` job stub (Ubuntu, parallel, no `needs:`) with `actionlint` step wired FIRST — guarantees every subsequent commit's workflow YAML is parseable before the more expensive preflight assertions land. Produced by **01-01-T1**.
- [x] `actionlint` binary available locally — one-line install via `go install github.com/rhysd/actionlint/cmd/actionlint@latest` OR `choco install actionlint` on the dev machine. Without this, the quick command cannot run locally and feedback latency degrades from 2s → 12min (dispatch-only). Noted in `01-01-PLAN.md` Task 1 "Sanity check before committing" guidance.

*Note:* No `pytest`/`jest`-style framework applies. The workflow's embedded preflight assertions + `lint-workflow` job collectively are the test harness. Wave 0 substrate = Plan 01's scaffold — no separate pre-wave-1 work required.

---

## Manual-Only Verifications

| Behavior | Requirement | Bound Task | Why Manual | Test Instructions |
|----------|-------------|------------|------------|-------------------|
| Negative-case preflight hard-fail | TC-02, TC-03 | 01-02-T3 | Cannot automate in CI without intentionally burning a red run; validates that the preflight ACTUALLY fails loud (not silent) when toolset out-of-band | Dispatch workflow via Actions UI with `msvc_toolset=14.01` (bogus). Expected: job fails at the MSVC probe step (Plan 02 Task 1 Step 4) with a named-step error like `"❌ MSVC toolset 14.01 not installable: component Microsoft.VisualStudio.Component.VC.14.01.x.y.x86.x64 not found via vswhere"`. Confirm: run does NOT reach `assert-msvc` or any other assertion step; Plan 03's `if: failure()` remediation-hint step writes a "Preflight FAILED — Remediation Context" block to `$GITHUB_STEP_SUMMARY` referencing upstream `#1543` and `runner-images#9701`. |
| Empirical `14.39` component availability on today's `windows-2022` image | TC-01, TC-02 | 01-02-T3 | Research RESEARCH.md Open Question #1 — runner-image state can only be probed at dispatch time due to `actions/runner-images#9701` VC component removal history | Dispatch with DEFAULT inputs (`msvc_toolset=14.39`). Expected green run end-to-end including Plan 03's forensics summary. If red at probe step: documents that `14.39` component is gone; triggers Open Question #3 — propose `14.40` + CUDA `12.6.1` fallback pair in a follow-up phase. Record outcome in `01-02-SUMMARY.md`. |
| Forensics summary readability | WF-01, DOC-04 | 01-03-T3 | Verifies the `$GITHUB_STEP_SUMMARY` output is actionable for a future on-call engineer, not just machine-green | After a green dispatch, open the run's `preflight` job Summary tab. Confirm: "Preflight Green — Toolchain Fingerprint" markdown table with all columns populated (MSVC toolset + `_MSC_VER`, nvcc version, CUDA path, Python, llama.cpp SHA, mamba version, Disk free, CPU) — no unexpanded `${{ steps.*.outputs.* }}` placeholders. After a red negative-case dispatch, confirm the "Preflight FAILED — Remediation Context" block is also present and actionable. |
| DOC-04 readability for on-call engineer | DOC-04 | 01-03-T3 | Verifies inline comments are readable as prose, not just grep-matched | Open workflow file in a text editor. Confirm: top-of-file preamble (~40 lines) covers fork rationale, _MSC_VER wall, what-this-workflow-does, roadmap, escape hatch, authoritative references. Inline comments at `msvc_toolset` input, `ilammy/msvc-dev-cmd` step, and `lint-workflow` job each reference upstream `#1543`. Mental test: could a future on-call engineer with zero context avoid re-enabling the compiler-bypass workaround? Confirm also: `grep -c -F 'allow-unsupported-compiler'` returns exactly `1` (ban-grep invariant preserved; every comment reference uses a paraphrase). |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies. Tasks 01-02-T3 and 01-03-T3 are `checkpoint:human-verify` — their automated verify (`gh run list ...`) is a post-condition check; the actual validation happens via human dispatch + Actions UI review. Documented in Manual-Only Verifications table above.
- [x] Sampling continuity: no 3 consecutive tasks without automated verify. 7 tasks total; 5 are fully automated (01-01-T1, 01-02-T1, 01-02-T2, 01-03-T1, 01-03-T2); 2 are human-verify checkpoints at wave terminations (01-02-T3, 01-03-T3). Longest gap is 1 task, not 3.
- [x] Wave 0 covers all MISSING references. Wave 0 substrate = Plan 01's scaffold (produced by 01-01-T1); every later task's automated command runs against the file that 01-01-T1 creates. Local `actionlint` binary dependency noted in Plan 01 Task 1 guidance.
- [x] No watch-mode flags. Quick command is single-shot `actionlint` + `grep -c -F ... | grep -q '^1$'`. Full command uses `gh run watch --exit-status` once per wave, not persistent.
- [x] Feedback latency < 12min. Quick = 2s local; full = ~10 runner-min per Windows dispatch per wave.
- [x] `nyquist_compliant: true` set in frontmatter. Every requirement ID (WF-01..05, TC-01..10, DOC-04 — 16 total) maps to at least one task in the Per-Task Verification Map above, confirmed by the Requirement → Task Coverage cross-check table.
- [x] Ban-grep invariant documented. Literal `allow-unsupported-compiler` permitted in exactly one location (Plan 01 Task 1's ban-grep command literal); Plan 03 Task 2 uses paraphrases for all DOC-04 comment-text references. Quick command enforces count-equals-1 via `| grep -q '^1$'` so any regression fails the per-task sampling check immediately.

**Approval:** approved 2026-04-15 (post plan-checker revision loop).
