---
phase: 3
slug: smoke-test-publish-gate
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-17
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | GitHub Actions workflow dispatch (empirical) |
| **Config file** | `.github/workflows/build-wheels-cuda-windows.yaml` |
| **Quick run command** | `actionlint .github/workflows/build-wheels-cuda-windows.yaml` |
| **Full suite command** | `gh workflow run build-wheels-cuda-windows.yaml` |
| **Estimated runtime** | ~15 minutes (full dispatch) |

---

## Sampling Rate

- **After every task commit:** Run `actionlint .github/workflows/build-wheels-cuda-windows.yaml`
- **After every plan wave:** Run `gh workflow run build-wheels-cuda-windows.yaml`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 900 seconds (workflow dispatch)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 03-01-01 | 01 | 1 | ST-01 | smoke/manual | `git ls-files llm_for_ci_test/tiny-llama.gguf` | Wave 0: commit file | pending |
| 03-01-02 | 01 | 1 | ST-02 | integration | `gh workflow run` + check job list | Wave 0: add job to YAML | pending |
| 03-01-03 | 01 | 1 | ST-03 | integration | Assert `Test-Path llama_cpp\__init__.py` = false in CI | Wave 0: add assertion step | pending |
| 03-01-04 | 01 | 1 | ST-04 | integration | `actions/download-artifact@v4` step succeeds in CI | Wave 0: add step | pending |
| 03-01-05 | 01 | 1 | ST-05 | integration | pip install exit code = 0 in CI | Wave 0: add step | pending |
| 03-01-06 | 01 | 1 | ST-06 | integration | Subprocess exit code = 0 in CI | Wave 0: add inference step | pending |
| 03-01-07 | 01 | 1 | ST-07 | integration | cuobjdump output contains sm_80,86,89,90 in CI | Wave 0: add cuobjdump step | pending |
| 03-01-08 | 01 | 1 | ST-08 | integration | Phase 4 grep-assert (deferred) | Phase 4 | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

- [ ] `llm_for_ci_test/tiny-llama.gguf` — already exists locally, needs `git add` + commit (ST-01)
- [ ] `smoke-test` job definition in workflow YAML — must be added (ST-02 through ST-07)
- [ ] All assertion steps — must be added to the workflow

*Note: ST-08 (publish gate assertion) is deferred to Phase 4.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Broken wheel causes red smoke-test | ST-06 | Requires deliberate spike branch with `-allow-unsupported-compiler` | Build a known-broken wheel, dispatch, verify smoke-test goes red and publish does not run |
| cuobjdump on Windows .dll | ST-07 | First-time validation — no prior example of cuobjdump on .dll | Inspect first dispatch log for `cuobjdump --list-elf` output on `ggml-cuda.dll` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 900s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
