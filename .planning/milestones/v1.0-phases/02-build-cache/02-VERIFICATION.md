---
phase: 02-build-cache
verified: 2026-04-16T00:00:00Z
status: passed
score: 9/9 must-haves verified
re_verification: false
human_verification:
  - test: "Dispatch the workflow and confirm wheel filename matches llama_cpp_python-0.3.20+cu126.ll<12-char-sha>-cp311-cp311-win_amd64.whl"
    expected: "Green run, wheel tag assertion passes, wheel size assertion passes, cuda-wheel artifact visible in run summary"
    why_human: "End-to-end build requires a live GitHub Actions runner with CUDA hardware access; cannot verify programmatically. SUMMARY.md documents a green dispatch producing a 358 MB cp311 wheel (commit 77f55fd), but the verifier cannot independently confirm runner state."
---

# Phase 2: Build & Cache Verification Report

**Phase Goal:** A dispatched run of the workflow produces a correctly-tagged cp311-cp311-win_amd64 wheel under 400 MB, with sccache and mamba caches wired for fast re-iteration, and post-build assertions that fail the job on tag or size regressions.
**Verified:** 2026-04-16
**Status:** passed (with one human-verification item for end-to-end dispatch confirmation)
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Build step activates MSVC via vcvarsall.bat in a clean pwsh subprocess using probe-msvc outputs, without ilammy | VERIFIED | Lines 652-675: $installPath/$selectedFull from steps.probe-msvc.outputs, cmd /c vcvarsall.bat x64, env diff applied to current session. No ilammy/msvc-dev-cmd anywhere outside comments. |
| 2 | CMAKE_ARGS env var contains Ninja generator, CUDA on, sccache launchers, /Z7 CMP0141, architecture targets, and CPU ISA disables | VERIFIED | Lines 708-724: -G Ninja, -DGGML_CUDA=on, CMAKE_CUDA_ARCHITECTURES=80-real;86-real;89-real;90-real;90-virtual, -DCMAKE_C/CXX/CUDA_COMPILER_LAUNCHER=sccache, -DCMAKE_POLICY_DEFAULT_CMP0141=NEW, -DCMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded, -DCMAKE_C/CXX_FLAGS_INIT="/Z7", -DGGML_AVX2/FMA/F16C=off |
| 3 | Wheel version embeds llama.cpp submodule SHA in PEP 440 local segment format (0.3.20+cu126.ll<sha12>) | VERIFIED | Lines 686-704: __init__.py read, baseVersion extracted via regex, $llamaSha12 = git -C vendor/llama.cpp rev-parse HEAD .Substring(0,12), fullVersion = "$baseVersion+$cudaTag.ll$llamaSha12". git checkout -- $initFile at line 744 restores the file. |
| 4 | sccache is installed via hendrikmuhs/ccache-action and wired as C/CXX/CUDA compiler launcher | VERIFIED | Line 633: hendrikmuhs/ccache-action@v1.2 with variant: sccache, append-timestamp: false, max-size: 500M. Lines 716-718: CMAKE_C/CXX/CUDA_COMPILER_LAUNCHER=sccache in CMAKE_ARGS. Grep confirms 3 occurrences of COMPILER_LAUNCHER=sccache. |
| 5 | Mamba pkgs cache uses split restore/save with if: always() for save-on-failure | VERIFIED | Line 604: actions/cache/restore@v4 (id: cache-mamba-restore). Line 824: actions/cache/save@v4 with if: always() (line 823) and key: ${{ steps.cache-mamba-restore.outputs.cache-primary-key }} (line 828). |
| 6 | Every new step has a timeout-minutes value | VERIFIED | grep count = 26 timeout-minutes declarations across entire file, covering all 24 steps in the build job plus the lint-workflow job steps. |
| 7 | Wheel tag regex assertion fails the job if tag is not cp311-cp311-win_amd64 | VERIFIED | Lines 754-784: $pyVer derived from inputs.python_version, $expectedPattern = "llama_cpp_python-.*-cp${pyVer}-cp${pyVer}-win_amd64\.whl$", throw on mismatch with targeted remediation hints. |
| 8 | Wheel size assertion fails the job if size >= 400 MB | VERIFIED | Lines 786-810: $maxSizeMB = 400, $sizeMB -ge $maxSizeMB throws with remediation. |
| 9 | Wheel is uploaded as a GitHub Actions artifact consumable by downstream smoke-test job | VERIFIED | Lines 812-820: actions/upload-artifact@v4, name: cuda-wheel, path: dist/*.whl, if-no-files-found: error, compression-level: 0, retention-days: 7. |

**Score:** 9/9 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `.github/workflows/build-wheels-cuda-windows.yaml` | Build body steps: VS BuildCustomizations copy, mamba cache restore, pip install build deps, sccache setup, vcvarsall+build, mamba cache save, sccache stats | VERIFIED | All 7 build body steps present at lines 549-837. python -m build --wheel at line 739 confirmed. |
| `.github/workflows/build-wheels-cuda-windows.yaml` | Post-build assertion steps (wheel tag, wheel size) and artifact upload step (upload-artifact@v4) | VERIFIED | Assert wheel tag at line 754, assert wheel size at line 786, upload-artifact@v4 at line 812. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| build step (vcvarsall activation) | steps.probe-msvc.outputs | install_path + selected_full_version to locate vcvarsall.bat | WIRED | Lines 652-653 consume ${{ steps.probe-msvc.outputs.install_path }} and ${{ steps.probe-msvc.outputs.selected_full_version }} |
| build step (version override) | 12-char SHA | git -C vendor/llama.cpp rev-parse HEAD (not assert-submodule 7-char output) | WIRED | Line 693: $llamaSha12 = (git -C vendor/llama.cpp rev-parse HEAD).Substring(0, 12) — re-derives 12-char SHA as documented |
| build step (CMAKE_ARGS) | sccache binary | CMAKE_C/CXX/CUDA_COMPILER_LAUNCHER=sccache | WIRED | Lines 716-718 in CMAKE_ARGS array; sccache binary installed by hendrikmuhs/ccache-action step preceding the build step |
| mamba cache save | mamba cache restore | cache-primary-key output from restore step | WIRED | Line 828: key: ${{ steps.cache-mamba-restore.outputs.cache-primary-key }} |
| wheel tag assertion step | build step wheel output | dist/*.whl glob matching regex pattern | WIRED | Lines 767-773: Get-ChildItem dist/*.whl, $wheelName -notmatch $expectedPattern |
| upload-artifact step | build step dist/ directory | path: dist/*.whl | WIRED | Line 817: path: dist/*.whl |
| forensics summary | build-wheel step outputs | steps.build-wheel.outputs.wheel_version and wheel_name | WIRED | Lines 871-872 reference ${{ steps.build-wheel.outputs.wheel_version }} and ${{ steps.build-wheel.outputs.wheel_name }} |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| BLD-01 | 02-01 | Wheel produced via python -m build --wheel using scikit-build-core | SATISFIED | Line 739: python -m build --wheel -C wheel.py-api="" |
| BLD-02 | 02-01 | sccache set up via hendrikmuhs/ccache-action@v1.2 with variant: sccache | SATISFIED | Line 633-639: ccache-action@v1.2, variant: sccache, append-timestamp: false |
| BLD-03 | 02-01 | sccache wired into CMake via C/CXX/CUDA COMPILER_LAUNCHER flags | SATISFIED | Lines 716-718: all three launchers in CMAKE_ARGS; confirmed by grep count of 3 |
| BLD-04 | 02-01 | MSVC debug info forced to /Z7 via CMP0141=NEW + Embedded | SATISFIED | Lines 719-722: CMAKE_POLICY_DEFAULT_CMP0141=NEW, CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded, C/CXX_FLAGS_INIT="/Z7" dual approach |
| BLD-05 | 02-01 | CUDA installer zip cached with if: always() | SATISFIED (reinterpreted) | Per CONTEXT.md line 33: "BLD-05 reinterpreted: since TC-05 locked CUDA install to mamba-only (no Jimver zip), BLD-05's intent is fulfilled by BLD-06 (mamba pkgs cache)." No dead CUDA installer zip cache needed. Mamba pkgs cache with if: always() covers the intent. |
| BLD-06 | 02-01 | Mamba/conda package directory cached with restore-keys fallback, saved if: always() | SATISFIED | Lines 604-610 (restore@v4), 822-828 (save@v4 with if: always()). Key includes cuda_version, runner.os, hashFiles of workflow. Restore-keys at line 610. |
| BLD-07 | 02-01 | VS CUDA integration extensions handled (per-run, not cached) | SATISFIED (reinterpreted) | CONTEXT.md decision: skip caching BuildCustomizations — just redo each run (<5s download+extract). Step at lines 549-600 downloads from Jimver link table, extracts with 7-Zip, copies to bc_path. Comment at line 562 documents the no-cache decision. |
| BLD-08 | 02-01 | Build uses Ninja generator with CMAKE_BUILD_PARALLEL_LEVEL set | SATISFIED | Line 709: -G Ninja in CMAKE_ARGS. Line 728: CMAKE_BUILD_PARALLEL_LEVEL=1 (changed from 2 after sccache daemon OOM on parallel nvcc, per fix commit e5ada5e). |
| BLD-09 | 02-01 | Each job step has a timeout-minutes bound | SATISFIED | 26 timeout-minutes declarations across the file. All 24 build job steps have explicit timeouts. Job-level timeout at line 125: 210 minutes (increased from 90 after dispatch empirical measurement). |
| BLD-10 | 02-02 | Wheel tagged correctly cp<pyver>-cp<pyver>-win_amd64; regex assertion guards against drift | SATISFIED | Lines 764-784: dynamic $pyVer from inputs.python_version, regex $expectedPattern, throw with targeted remediation hints for py3-none/abi3/none-any mismatches. |
| BLD-11 | 02-02 | Wheel size under 400 MB; CI fails if exceeded | SATISFIED | Lines 793-809: $maxSizeMB = 400, $sizeMB -ge $maxSizeMB throws. Verified by dispatch at 358 MB per SUMMARY. |
| BLD-12 | 02-01 | Wheel version embeds llama.cpp submodule short SHA (0.3.20+cu126.ll<sha>) | SATISFIED | Lines 686-704: 12-char SHA derived, fullVersion = "$baseVersion+$cudaTag.ll$llamaSha12", __init__.py rewritten then restored. |
| BLD-13 | 02-02 | Wheel uploaded as GitHub Actions artifact via upload-artifact@v4 | SATISFIED | Lines 812-820: upload-artifact@v4, name: cuda-wheel, path: dist/*.whl, if-no-files-found: error, compression-level: 0, retention-days: 7. |

**All 13 BLD-* requirements: SATISFIED**

No orphaned requirements — all 13 BLD-01..BLD-13 IDs appear in plans 02-01 and 02-02.

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| None detected | — | — | — |

Scanned `.github/workflows/build-wheels-cuda-windows.yaml` for TODO/FIXME/XXX/HACK/PLACEHOLDER, empty implementations, and stub patterns. Zero hits. The workflow is substantive end-to-end.

Notable deviations from the original plan that were fixed during dispatch verification and are correctly reflected in the final file:

- SKBUILD_WHEEL_PY_API env var replaced with -C wheel.py-api="" config-setting (scikit-build-core ignores empty string env vars)
- CMAKE_BUILD_PARALLEL_LEVEL lowered from 2 to 1 (parallel nvcc killed sccache daemon)
- Build step timeout increased from 60 to 180 min; job timeout from 90 to 210 min
- VS BuildCustomizations source changed from mamba (which does not ship MSBuildExtensions) to Jimver CUDA installer download + 7-Zip extraction
- sccache append-timestamp: false added to prevent per-run key instability

All are self-consistent with the final workflow state and documented in 02-02-SUMMARY.md.

### Human Verification Required

#### 1. End-to-End Dispatch Green

**Test:** Run `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.6.3` and wait for completion.
**Expected:**
- All build steps pass
- Wheel filename matches `llama_cpp_python-0.3.20+cu126.ll<12-char-sha>-cp311-cp311-win_amd64.whl`
- Wheel tag assertion step green
- Wheel size assertion step green (< 400 MB)
- cuda-wheel artifact visible in run artifacts
- sccache stats show statistics
- mamba cache save step ran with if: always()
- Forensics summary includes wheel version and filename rows

**Why human:** Requires a live GitHub Actions windows-2022 runner with CUDA toolkit accessible. Cannot verify programmatically from source. SUMMARY.md documents a successful dispatch producing a 358 MB cp311 wheel across 6 fix iterations, with the final commit 77f55fd (sccache append-timestamp fix) producing a confirmed green run. However, runner image rotation or environment drift could affect a future dispatch. This is flagged as expected human validation per the checkpoint:human-verify task in 02-02-PLAN.md.

### Summary

Phase 2 goal is structurally achieved. All 9 observable truths are verified in the codebase, all 13 BLD-* requirements are satisfied, all 7 key links are wired, and no anti-patterns are present.

**Key structural facts verified:**

1. The workflow file has 24 job steps ordered correctly: Phase 1 preflight (steps 1-12), Phase 2 build body (steps 13-22: VS BuildCustomizations download, mamba cache restore, pip install, sccache setup, MSVC+build, tag assertion, size assertion, upload, mamba cache save, sccache stats), then forensics summary and remediation hint as final steps.

2. The vcvarsall activation pattern (env-before/cmd-subprocess/env-diff) is implemented inline in the build step — correct, because GH Actions environment does not persist across steps.

3. The -C wheel.py-api="" config-setting on python -m build is the correct override for scikit-build-core (not the env var SKBUILD_WHEEL_PY_API which scikit-build-core ignores when empty). This is the critical fix that produces cp311-cp311 tag instead of py3-none.

4. BLD-05 and BLD-07 deviate from their original spec wording but are correctly reinterpreted per CONTEXT.md: BLD-05's CUDA installer zip cache intent is covered by BLD-06's mamba pkgs cache; BLD-07's VS integration cache is intentionally omitted because the per-run download takes <5 seconds.

5. The ban-grep invariant is preserved: `allow-unsupported-compiler` does not appear in the workflow outside the lint-workflow ban-grep step itself (which uses character class escaping to avoid self-match).

6. Job timeout is 210 minutes (not the plan's original 90), reflecting empirical measurement: cold CUDA build with 5 arch targets and parallel=1 takes ~120 min.

---

_Verified: 2026-04-16_
_Verifier: Claude (gsd-verifier)_
