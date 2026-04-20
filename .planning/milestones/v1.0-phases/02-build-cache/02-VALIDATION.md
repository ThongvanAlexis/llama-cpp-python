---
phase: 2
slug: build-cache
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-16
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | GitHub Actions workflow dispatch + PowerShell assertions (same as Phase 1) |
| **Config file** | `.github/workflows/build-wheels-cuda-windows.yaml` (self-contained) |
| **Quick run command** | `actionlint .github/workflows/build-wheels-cuda-windows.yaml` |
| **Full suite command** | `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.6.3 && gh run watch --exit-status` |
| **Estimated runtime** | ~98 minutes (cold build); ~30 minutes (warm sccache) |

---

## Sampling Rate

- **After every task commit:** Run `actionlint .github/workflows/build-wheels-cuda-windows.yaml`
- **After every plan wave:** Full dispatch + `gh run watch --exit-status`
- **Before `/gsd:verify-work`:** Full suite green; sccache stats showing >30% hit rate on warm rebuild
- **Max feedback latency:** ~5 seconds (actionlint); ~98 minutes (full dispatch)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 02-01-01 | 01 | 1 | BLD-01 | dispatch | `gh run watch --exit-status` (build step green) | Wave 0 | pending |
| 02-01-02 | 01 | 1 | BLD-02 | dispatch | sccache setup step green + `sccache --show-stats` in log | Wave 0 | pending |
| 02-01-03 | 01 | 1 | BLD-03 | dispatch | grep CMAKE_ARGS for COMPILER_LAUNCHER in workflow | Wave 0 | pending |
| 02-01-04 | 01 | 1 | BLD-04 | dispatch | sccache stats show >0 cache hits on rebuild | Wave 0 | pending |
| 02-01-05 | 01 | 1 | BLD-05 | dispatch | Fulfilled by BLD-06 per CONTEXT.md | Wave 0 | pending |
| 02-01-06 | 01 | 1 | BLD-06 | dispatch | Cache restore/save steps in log + retry shows restore | Wave 0 | pending |
| 02-01-07 | 01 | 1 | BLD-07 | dispatch | BuildCustomizations copy step green | Wave 0 | pending |
| 02-01-08 | 01 | 1 | BLD-08 | dispatch | CMAKE_ARGS contains `-G Ninja` + parallel level logged | Wave 0 | pending |
| 02-01-09 | 01 | 1 | BLD-09 | lint | `grep -c 'timeout-minutes' workflow` >= expected count | Wave 0 | pending |
| 02-01-10 | 01 | 1 | BLD-10 | dispatch | Tag assertion step green (regex match) | Wave 0 | pending |
| 02-01-11 | 01 | 1 | BLD-11 | dispatch | Size assertion step green | Wave 0 | pending |
| 02-01-12 | 01 | 1 | BLD-12 | dispatch | Wheel filename contains `+cu126.ll` segment | Wave 0 | pending |
| 02-01-13 | 01 | 1 | BLD-13 | dispatch | upload-artifact step green | Wave 0 | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.* Phase 1 workflow infrastructure provides the substrate. Phase 2 appends steps to the existing preflight job (or adds a new build job with `needs: [preflight]`). All verification is via CI dispatch and actionlint.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| sccache warm rebuild >30% hit rate | BLD-04 | Requires two sequential dispatches — cannot automate in single CI run | 1. Run dispatch (cold build). 2. Run dispatch again (warm). 3. Check sccache stats in logs for >30% hit rate |
| Save-on-failure caches persist | BLD-05/06 | Requires deliberately-failing build + retry | 1. Introduce compile error. 2. Run dispatch (build fails). 3. Check cache save steps ran. 4. Fix error. 5. Run dispatch again. 6. Verify cache restore in logs |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 98 minutes (dispatch) / < 5s (lint)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
