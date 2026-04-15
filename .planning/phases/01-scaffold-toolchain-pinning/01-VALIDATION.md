---
phase: 1
slug: scaffold-toolchain-pinning
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-15
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | actionlint (workflow static analysis) + PowerShell `Select-String` (ban-grep) + dispatched GitHub Actions run (`build-wheels-cuda-windows.yaml` preflight-only) |
| **Config file** | `.github/workflows/build-wheels-cuda-windows.yaml` (self-contained; `lint-workflow` job IS the local-reproducible test harness) |
| **Quick run command** | `actionlint .github/workflows/build-wheels-cuda-windows.yaml && Select-String -Path .github/workflows/build-wheels-cuda-windows.yaml -Pattern 'allow-unsupported-compiler' -SimpleMatch; if ($LASTEXITCODE -eq 0) { throw 'BANNED flag found' }` |
| **Full suite command** | `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.4.1 -f msvc_toolset=14.39 && gh run watch --exit-status` (watches latest preflight dispatch to green) |
| **Estimated runtime** | Quick: ~2s (actionlint + ban-grep). Full: ~8–12 min (Windows runner cold-start + mamba CUDA install + preflight assertions; no compile in this phase) |

---

## Sampling Rate

- **After every task commit:** Run quick command (`actionlint` + ban-grep). Workflow must parse and must not contain `allow-unsupported-compiler`.
- **After every plan wave:** Run quick command. (Full dispatch reserved for wave completion only — it costs ~10 runner minutes.)
- **Before `/gsd:verify-work`:** Full suite must be green — one successful `workflow_dispatch` run reaching the `preflight` job's terminal `echo "✅ Toolchain preflight passed"` step with green status on BOTH `lint-workflow` and `preflight` jobs.
- **Max feedback latency:** Quick = 2s (local). Full = 12 min (dispatch). Acceptable because the `lint-workflow` job re-runs the quick checks on every push/PR — catches regressions without manual dispatch.

---

## Per-Task Verification Map

*Plan/task IDs are pre-planning placeholders based on the research's suggested wave structure (Wave 0: scaffold skeleton, Wave 1: toolchain install + preflight assertions, Wave 2: `lint-workflow` job + forensics summary). Planner may adjust granularity; this map MUST be refreshed during plan revision to bind to actual task IDs.*

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 0 | WF-01, WF-02 | workflow-parse | `actionlint .github/workflows/build-wheels-cuda-windows.yaml` (exit 0 ⇒ valid YAML + dispatch inputs resolve) | ❌ W0 | ⬜ pending |
| 1-01-02 | 01 | 0 | WF-03 | dispatch-inputs | `gh workflow view build-wheels-cuda-windows.yaml --yaml \| yq '.on.workflow_dispatch.inputs \| keys'` must list `python_version`, `cuda_version`, `msvc_toolset` with correct defaults | ❌ W0 | ⬜ pending |
| 1-01-03 | 01 | 0 | WF-04 | trigger-exclusivity | `yq '.on \| keys' .github/workflows/build-wheels-cuda-windows.yaml` returns `["workflow_dispatch"]` only (no `push`/`pull_request`/`schedule`) | ❌ W0 | ⬜ pending |
| 1-01-04 | 01 | 1 | TC-01, TC-02 | msvc-activation | `ilammy/msvc-dev-cmd@v1` step with `toolset: ${{ inputs.msvc_toolset }}` present; preflight asserts `cl.exe /Bv` prints `19.39.` prefix | ❌ W0 | ⬜ pending |
| 1-01-05 | 01 | 1 | TC-03 | msc-ver-cap | Preflight runs `cl.exe 2>&1 \| Select-String '_MSC_VER'` equivalent and enforces `_MSC_VER <= 1900 + minor(msvc_toolset)` (derivation: `$cap = 1900 + [int](($env:MSVC_TOOLSET -split '\.')[1])`); step must FAIL JOB on violation via `exit 1` | ❌ W0 | ⬜ pending |
| 1-01-06 | 01 | 1 | TC-04, TC-05 | cuda-version | Preflight runs `nvcc --version` and asserts output contains `${{ inputs.cuda_version }}`; single `where nvcc` path check | ❌ W0 | ⬜ pending |
| 1-01-07 | 01 | 1 | TC-06 | cuda-unification | Preflight asserts `(where nvcc).Count -eq 1` AND `Get-ChildItem env: \| Where-Object Name -match '^CUDA_PATH_V'` returns empty | ❌ W0 | ⬜ pending |
| 1-01-08 | 01 | 1 | TC-07 | mamba-only-cuda | Only one CUDA install step (mamba `cuda-toolkit`); grep for `Jimver/cuda-toolkit@` OR `nvcc_*.exe --silent` returns zero matches | ❌ W0 | ⬜ pending |
| 1-01-09 | 01 | 1 | TC-08 | vs-buildcustomizations | PowerShell block uses `vswhere -products * -latest -property installationPath` to discover dynamically; strict-mode `if (-not $vsPath) { throw }` present; no hardcoded `2019\Enterprise` path | ❌ W0 | ⬜ pending |
| 1-01-10 | 01 | 1 | TC-09 | strict-mode | Every `run: \|` with `pwsh` shell starts with `Set-StrictMode -Version Latest` AND `$ErrorActionPreference = 'Stop'` (grep-enforced in `lint-workflow`) | ❌ W0 | ⬜ pending |
| 1-01-11 | 01 | 1 | TC-10 | named-step-errors | Every preflight assertion is a named step (has `name:` field) with explicit `exit 1` or `throw` on failure; no bare inline assertions that would show as "step N" in logs | ❌ W0 | ⬜ pending |
| 1-01-12 | 01 | 2 | WF-05, DOC-04 | ban-grep-selfcheck | `lint-workflow` job (Ubuntu, parallel, no `needs:`): `grep -F 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml` exits non-zero (zero matches = pass); step MUST fail job on any match | ❌ W0 | ⬜ pending |
| 1-01-13 | 01 | 2 | DOC-04 | actionlint-job | `lint-workflow` job runs `rhysd/actionlint@v1` (or equivalent); any schema/expression error fails the job | ❌ W0 | ⬜ pending |
| 1-01-14 | 01 | 2 | WF-01, WF-02, TC-10 | forensics-summary | On failure, preflight writes diagnostic block to `$env:GITHUB_STEP_SUMMARY` (toolset probed, `cl.exe /Bv` output, `nvcc --version`, `where nvcc`, remediation hint); verify with `if: failure()` step present | ❌ W0 | ⬜ pending |
| 1-01-15 | 01 | 2 | TC-02, TC-03 | negative-case-probe | Dispatch with `msvc_toolset=14.01` (intentionally bogus) MUST cause preflight to fail at the probe step with named error (not silently install & segfault) — documented in task as manual smoke test (see Manual-Only section) | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

This phase authors a GitHub Actions YAML file — there is no traditional test framework to install. "Wave 0" here = skeleton-scaffold wave that provides the parse/lint/ban-grep substrate every later task verifies against.

- [ ] `.github/workflows/build-wheels-cuda-windows.yaml` — skeleton with `on: workflow_dispatch` + three inputs (`python_version`, `cuda_version`, `msvc_toolset`) and empty `preflight` job stub. Required before tasks 1-01-04..14 can run their automated checks.
- [ ] `lint-workflow` job stub (Ubuntu, parallel, no `needs:`) with `actionlint` step wired FIRST — guarantees every subsequent commit's workflow YAML is parseable before the more expensive preflight assertions land.
- [ ] `actionlint` binary available locally — one-line install via `go install github.com/rhysd/actionlint/cmd/actionlint@latest` OR `choco install actionlint` on the dev machine. Without this, the quick command cannot run locally and feedback latency degrades from 2s → 12min (dispatch-only).

*Note:* No `pytest`/`jest`-style framework applies. The workflow's embedded preflight assertions + `lint-workflow` job collectively are the test harness.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Negative-case preflight hard-fail | TC-02, TC-03 | Cannot automate in CI without intentionally burning a red run; validates that the preflight ACTUALLY fails loud (not silent) when toolset out-of-band | Dispatch workflow via Actions UI with `msvc_toolset=14.01` (bogus). Expected: job fails at the MSVC probe step with a named-step error like `"❌ MSVC toolset 14.01 not installable: component Microsoft.VisualStudio.Component.VC.14.01.x.y.x86.x64 not found via vswhere"`. Confirm: run does NOT reach `nvcc` install step; `$GITHUB_STEP_SUMMARY` contains remediation hint. |
| Empirical `14.39` component availability on today's `windows-2022` image | TC-01 | Research RESEARCH.md Open Question #1 — runner-image state can only be probed at dispatch time due to `actions/runner-images#9701` VC component removal | Dispatch with DEFAULT inputs (`msvc_toolset=14.39`). Expected green run. If red at probe step: documents that `14.39` component is gone; triggers Open Question #3 — propose `14.40` + CUDA `12.6.1` fallback pair in a follow-up phase. |
| Forensics summary readability | WF-01, DOC-04 | Verifies the `$GITHUB_STEP_SUMMARY` output is actionable for a future on-call engineer, not just machine-green | After a red negative-case dispatch, open the run's Summary tab. Confirm: dispatched `msvc_toolset`, `cl.exe /Bv` snippet, `nvcc --version` snippet, `where nvcc` output, and remediation hint are present as a markdown block. |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies (tasks 1-01-15 is manual-only — documented in Manual-Only section with dispatch instructions)
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify (14 of 15 tasks have automated verify; only 1-01-15 is manual — no 3-in-a-row gap)
- [ ] Wave 0 covers all MISSING references (skeleton YAML + `lint-workflow` stub + local `actionlint` binary = all Wave 1/2 checks runnable)
- [ ] No watch-mode flags (quick command is single-shot `actionlint` + ban-grep; full command uses `gh run watch --exit-status` once per wave, not persistent)
- [ ] Feedback latency < 12min (quick = 2s local; full = ~10 runner-min one Windows dispatch per wave)
- [ ] `nyquist_compliant: true` set in frontmatter (toggle after planner binds real task IDs in revision loop and plan-checker confirms every requirement ID WF-01..05, TC-01..10, DOC-04 maps to at least one row above)

**Approval:** pending
