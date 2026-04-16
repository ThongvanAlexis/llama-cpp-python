---
phase: 01-scaffold-toolchain-pinning
verified: 2026-04-16T18:00:00Z
status: gaps_found
score: 4/5 success criteria verified
gaps:
  - truth: "On a green preflight run the job log shows cl.exe /Bv reporting _MSC_VER <= 1940 AND nvcc --version matches the dispatched cuda_version; if either is out of range the job fails with a clear named-step error"
    status: partial
    reason: "TC-03 requirement specifies ilammy/msvc-dev-cmd@v1 must activate the toolset. The actual implementation dropped ilammy (Plan 02 empirical deviation) and uses cl.exe by full path instead. The assertion logic is correct, but the requirement text says 'ilammy/msvc-dev-cmd@v1 with toolset: steps.probe-msvc.outputs.selected_toolset' and ilammy is absent from the file as an active step. The REQUIREMENTS.md TC-03 still reads 'activated via ilammy/msvc-dev-cmd@v1' — which is marked [x] complete but the [x] in REQUIREMENTS.md does not reflect the actual implementation. The preflight _MSC_VER assertion (TC-04) IS correctly implemented via cl.exe full path. The step comment acknowledges the deviation. This is a documentation gap in REQUIREMENTS.md, not a functional failure, but the requirement as written is not satisfied by the implementation."
    artifacts:
      - path: ".github/workflows/build-wheels-cuda-windows.yaml"
        issue: "No step uses ilammy/msvc-dev-cmd@v1 — it is referenced only in comments. TC-03 in REQUIREMENTS.md says 'activated via ilammy/msvc-dev-cmd@v1'. The plan's deviation was intentional (vcvarsall overflow) but REQUIREMENTS.md was not updated to reflect the new activation mechanism."
    missing:
      - "Either update TC-03 in REQUIREMENTS.md to say 'MSVC toolset activated via cmake VS generator or vcvarsall subprocess (ilammy dropped due to vcvarsall INCLUDE/LIB overflow with conda — see Plan 02 SUMMARY decision 4)' OR acknowledge TC-03 is superseded by the cl.exe full-path approach in Phase 2 build context"
human_verification:
  - test: "Dispatch workflow with default inputs and open Actions UI Summary tab for the preflight job"
    expected: "A 'Preflight Green — Toolchain Fingerprint' markdown table appears with all 11 columns populated (no unexpanded ${{ steps.*.outputs.* }} expressions). Log ends with 'Toolchain preflight passed'."
    why_human: "The forensics summary step (step 13) writes to $GITHUB_STEP_SUMMARY — cannot verify rendered output without a real dispatch. The step code exists and wires all step output IDs correctly, but formatting issues or unexpanded expressions only surface in the Actions UI."
  - test: "Dispatch with msvc_toolset=14.01 and inspect probe-msvc step log"
    expected: "Job fails at step 1 (Probe and auto-select MSVC toolset) within ~5 seconds with a remediation hint block referencing runner-images#9701 and upstream #1543. The run does NOT reach assert-msvc."
    why_human: "Negative-case hard-fail behavior can only be confirmed against the live runner. The code logic is verifiable (the throw path exists with the right conditions), but whether it fires correctly on the actual runner environment requires a dispatch."
---

# Phase 1: Scaffold & Toolchain Pinning Verification Report

**Phase Goal:** A `workflow_dispatch`-only Windows workflow file exists that, when dispatched with default inputs, reliably activates MSVC 14.40 and CUDA 12.6.3 on a `windows-2022` runner, with preflight assertions that fail loudly (rather than silently producing segfaulting wheels) when the toolchain is out of band.

**Verified:** 2026-04-16T18:00:00Z
**Status:** gaps_found (1 requirement documentation gap + 2 human-verify items)
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A dispatcher can run `build-wheels-cuda-windows.yaml` from the Actions UI with `python_version=3.11` and `cuda_version=12.6.3` defaults, and the workflow reaches the preflight checks without YAML/trigger errors. | VERIFIED | `on: workflow_dispatch` is sole trigger; 5 inputs present with correct defaults; YAML parses (654 lines); lint-workflow job with actionlint provides per-dispatch parse validation. |
| 2 | On a green preflight run the job log shows `cl.exe /Bv` reporting `_MSC_VER <= 1940` AND `nvcc --version` matches the dispatched `cuda_version`; if either is out of range the job fails with a clear named-step error. | PARTIAL | `assert-msvc` step implements the _MSC_VER assertion via cl.exe full path (correct). `assert-nvcc` step implements nvcc version check (correct). BUT TC-03 in REQUIREMENTS.md says ilammy/msvc-dev-cmd@v1 must activate the toolset — ilammy is not present as an active step (dropped in Plan 02 due to vcvarsall overflow). The assertion logic is sound; the stated activation mechanism in the requirement is not. |
| 3 | Grepping the workflow file for the literal string `allow-unsupported-compiler` returns zero matches, and a CI self-check step enforces this. | VERIFIED | `grep -c -F 'allow-unsupported-compiler'` returns 0. The ban-grep step uses regex char class `allow[-]unsupported[-]compiler` to avoid self-match (Plan 02 fix 23ce327). The `lint-workflow` job enforces this on every dispatch. Zero literal occurrences in comments (all references paraphrased). |
| 4 | Only one CUDA install path is active in the job (mamba `cuda-toolkit`); `where nvcc` returns exactly one path and no stray `CUDA_PATH_V*_*` env variables survive into the build step. | VERIFIED | Step 6 installs via `mamba install ... cuda-toolkit`; no Jimver step present. `assert-cuda-single-path` step sweeps `CUDA_PATH_V*_*` env vars and asserts exactly one nvcc path under `$env:CONDA_PREFIX`. Code logic correct. |
| 5 | The VS BuildCustomizations directory is discovered dynamically and the workflow fails loudly (strict-mode) if no VS 2022 install is found, rather than silently no-op'ing. | VERIFIED | `discover-vs` step uses `vswhere -latest -property installationPath`; throws with remediation hint if empty. Dynamic glob `MSBuild\Microsoft\VC\*\BuildCustomizations` with strict-mode `throw` on zero results. No hardcoded `2019\Enterprise` path. |

**Score:** 4/5 truths fully verified (1 partial — TC-03 implementation deviation)

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `.github/workflows/build-wheels-cuda-windows.yaml` | Two-job workflow file with 5 inputs, preflight job (14 steps), lint-workflow job (3 steps) | VERIFIED | 654 lines. `on: workflow_dispatch` only. 5 inputs: `python_version` (choice, default `3.11`), `cuda_version` (choice, default `12.6.3`), `msvc_toolset` (string, default `auto`), `verbose` (boolean), `dry_run` (boolean). `preflight` on `windows-2022`, timeout 20 min, shell pwsh. 14 named steps. `lint-workflow` on `ubuntu-latest`, 3 steps, no `needs:`. `permissions: contents: read`. |
| `.github/workflows/build-wheels-cuda.yaml` (upstream) | Untouched | VERIFIED | `git diff HEAD .github/workflows/build-wheels-cuda.yaml` returns empty. WF-01 preserved. |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `lint-workflow` ban-grep step | Workflow file itself | `grep -nHE 'allow[-]unsupported[-]compiler'` scoped to `$WORKFLOW` only | VERIFIED | Ban-grep targets `.github/workflows/build-wheels-cuda-windows.yaml` only (not upstream file, not repo-wide). Regex char class avoids self-match. |
| `assert-msvc` step | `inputs.msvc_toolset` / `probe-msvc` outputs | `1900 + [int]$parts[1]` formula computed from `steps.probe-msvc.outputs.selected_toolset` | VERIFIED | Formula present 4 times. No hardcoded `1939` or `1944` constants. Cap derived from CUDA compat matrix hashtable via `probe-msvc` step. |
| Forensics summary step (step 13) | Plan 02 assertion step outputs | `${{ steps.assert-msvc.outputs.msc_ver }}`, `${{ steps.assert-nvcc.outputs.nvcc_version }}`, `${{ steps.assert-submodule.outputs.llama_sha }}`, `${{ steps.assert-python.outputs.python_version }}`, `${{ steps.discover-vs.outputs.vs_install_path }}` | VERIFIED | All 5 output references present in forensics step body. `Out-File -Append -Encoding utf8` on `$GITHUB_STEP_SUMMARY`. Terminal `Write-Host "Toolchain preflight passed"` present. |
| `probe-msvc` step | `assert-msvc` step | `steps.probe-msvc.outputs.{selected_toolset, selected_full_version, install_path, msc_ver_cap}` | VERIFIED | Probe emits 4 outputs via `>> $env:GITHUB_OUTPUT`. `assert-msvc` consumes all 4. cl.exe full path constructed from `$installPath\VC\Tools\MSVC\$selectedFull\bin\HostX64\x64\cl.exe`. |
| DOC-04 top-of-file preamble | Upstream issues | `#1543` (9 occurrences), `runner-images#9701` (2 occurrences) | VERIFIED | Top-of-file preamble ~76 lines. Covers: fork rationale, _MSC_VER/nvcc compat wall, what-this-workflow-does, roadmap, escape hatch, authoritative references. All banned-flag references paraphrased. |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| WF-01 | 01-01 | Separate workflow file, not upstream's | SATISFIED | File exists at `.github/workflows/build-wheels-cuda-windows.yaml`. Upstream `build-wheels-cuda.yaml` has zero diff. |
| WF-02 | 01-01 | `workflow_dispatch` only trigger | SATISFIED | `on:` block has exactly `workflow_dispatch` key. No push/PR/release/schedule. |
| WF-03 | 01-01 | `python_version` input, default `3.11` | SATISFIED | Input present: `type: choice`, `default: '3.11'`, options `['3.10','3.11','3.12']`. |
| WF-04 | 01-01 | `cuda_version` input, default `12.6.3` | SATISFIED | Input present: `type: choice`, `default: '12.6.3'`, options `['12.6.3']`. (Bumped from `12.4.1` per OQ1 resolution 2026-04-15.) |
| WF-05 | 01-02 | Checkout with `submodules: recursive` | SATISFIED | Step 3 (`Checkout (with submodules)`) uses `actions/checkout@v4` with `submodules: recursive`. `assert-submodule` step verifies `vendor/llama.cpp/CMakeLists.txt` exists at runtime. |
| TC-01 | 01-01 | Runner pinned to `windows-2022` | SATISFIED | `preflight.runs-on: windows-2022` (literal string, never `windows-latest`). Verified in YAML. |
| TC-02 | 01-02 | MSVC auto-select from CUDA compat matrix | SATISFIED | `probe-msvc` step enumerates `VC\Tools\MSVC\*` from disk, looks up CUDA major.minor in `$cudaMsvcCap` hashtable, picks newest toolset where `_MSC_VER <= cap`. Override mode validates installed AND compatible. |
| TC-03 | 01-02 | MSVC toolset activated via `ilammy/msvc-dev-cmd@v1` | DOCUMENTATION GAP | REQUIREMENTS.md says `ilammy/msvc-dev-cmd@v1` with `toolset: steps.probe-msvc.outputs.selected_toolset`. ilammy was dropped in Plan 02 (vcvarsall INCLUDE/LIB overflow with conda). Phase 1 calls `cl.exe` by full path for _MSC_VER verification. REQUIREMENTS.md TC-03 marked [x] complete but text has not been updated to reflect the actual implementation (cl.exe full path / no vcvarsall activation in Phase 1; Phase 2 build will use cmake VS generator). The functional intent of TC-03 (toolset activated correctly for build) is not yet realized at all since Phase 2 hasn't landed — but the Phase 1 preflight correctly identifies and validates the toolset. |
| TC-04 | 01-02 | Preflight asserts _MSC_VER pin integrity + CUDA compat | SATISFIED | `assert-msvc` step: (1) pin integrity — `$actualMscVer -ne $expectedMscVer` throws, (2) CUDA compat — `$actualMscVer -gt $mscVerCap` throws. Both use `probe-msvc` outputs. Cap computed from `1900 + [int]$parts[1]`. |
| TC-05 | 01-02 | Single CUDA install path (mamba) | SATISFIED | Step 6 uses `mamba install ... cuda-toolkit`. No `Jimver/cuda-toolkit` action present (0 occurrences). |
| TC-06 | 01-02 | Preflight asserts `nvcc --version` matches `cuda_version` | SATISFIED | `assert-nvcc` step: regex matches `release (\d+\.\d+)` from `nvcc --version`, compares to major.minor of `inputs.cuda_version`. Throws with remediation hint on mismatch. |
| TC-07 | 01-02 | `CUDA_PATH` and `CUDA_PATH_V*_*` normalized | SATISFIED | `assert-cuda-single-path` step sweeps `^CUDA_PATH_V\d+_\d+$` env vars and removes them. Asserts `where.exe nvcc` returns exactly 1 path under `$env:CONDA_PREFIX`. |
| TC-08 | 01-02 | VS BuildCustomizations path discovered dynamically | SATISFIED | `discover-vs` step uses `vswhere -latest -property installationPath` + glob `MSBuild\Microsoft\VC\*\BuildCustomizations`. Throws with strict-mode if no VS found. No hardcoded edition paths. |
| TC-09 | 01-02 | `LongPathsEnabled=1` before checkout | SATISFIED | Step 2 (`Enable Windows long paths`) sets registry key and verifies with `Get-ItemProperty`. Runs at position 2, BEFORE step 3 (checkout). |
| TC-10 | 01-01 | No `allow-unsupported-compiler` in workflow; CI self-check | SATISFIED | `grep -c -F 'allow-unsupported-compiler'` returns 0. Ban-grep step in `lint-workflow` uses regex char class to avoid self-match. Runs on every dispatch. Zero literal occurrences in any context. |
| DOC-04 | 01-03 | Inline comments at 4 locked sites, referencing #1543 | SATISFIED | (1) Top-of-file preamble 76 lines covering all required topics; (2) `msvc_toolset` input comment with paraphrased banned-flag reference; (3) comment above ilammy note explaining no-ilammy rationale; (4) `lint-workflow` job header comment explaining ban-grep scope. `#1543` appears 9 times. `runner-images#9701` appears 2 times. |

**Orphaned requirements check:** No requirements mapped to Phase 1 in REQUIREMENTS.md traceability table are missing from this coverage. All 16 IDs (WF-01..05, TC-01..10, DOC-04) accounted for.

---

## Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| `.github/workflows/build-wheels-cuda-windows.yaml` line 371 | `"This means ilammy/msvc-dev-cmd did NOT activate the requested toolset correctly"` — error message refers to ilammy activating the toolset, but ilammy is not used | Info | Misleading error message if `assert-msvc` ever fires in Phase 1. The check verifies `cl.exe` reports the expected `_MSC_VER` — if it doesn't match, the message incorrectly blames ilammy. This is a cosmetic issue; the assertion logic is correct. |
| `.github/workflows/build-wheels-cuda-windows.yaml` | `ilammy` referenced 7 times in comments and error messages but never as an active `uses:` step | Info | Creates documentation/implementation inconsistency. The `NOTE: ilammy/msvc-dev-cmd is NOT used here` comment at line 323 correctly explains the deviation, but the `assert-msvc` error message (line 371) is stale. |
| REQUIREMENTS.md TC-03 | `[x]` marked complete while text still says `ilammy/msvc-dev-cmd@v1` | Warning | A reader of REQUIREMENTS.md will believe ilammy is in the workflow. Plan 02 SUMMARY documents the deviation clearly, but REQUIREMENTS.md itself is misleading. |

---

## Human Verification Required

### 1. Forensics Summary Tab Rendering

**Test:** Dispatch the workflow from the Actions UI with default inputs (`python_version=3.11`, `cuda_version=12.6.3`, `msvc_toolset=auto`). After the run completes green, open the `preflight` job's Summary tab (not the log).

**Expected:** A markdown table titled "Preflight Green — Toolchain Fingerprint" renders with all 11 rows populated: Runner OS, VS install path, VS BuildCustomizations, MSVC toolset (with `_MSC_VER=1944`), nvcc version (`12.6`), CUDA path (showing conda env path), Python (`3.11` at conda env), llama.cpp SHA (7-char), mamba version, disk free (C:), CPU. No unexpanded `${{ steps.*.outputs.* }}` placeholders. The log ends with `Toolchain preflight passed`.

**Why human:** The forensics step writes to `$GITHUB_STEP_SUMMARY` which only renders in the Actions UI. The code correctly wires all 5 step output references (verified programmatically), but expression expansion and markdown rendering require a live dispatch. The 01-03-SUMMARY.md documents a successful dispatch with these results (2026-04-16) but this occurred before VERIFICATION.md was created.

### 2. Negative-Case Hard-Fail Behavior

**Test:** Dispatch the workflow with `msvc_toolset=14.01` (a toolset version that is not installed on windows-2022).

**Expected:** The `probe-msvc` step (step 1) fails within ~5 seconds with a named-step error showing "Requested MSVC toolset '14.01' is NOT installed on this windows-2022 image" along with available toolset pins and remediation links. The job MUST NOT reach `assert-msvc` (step 7) or any later step. The `lint-workflow` job must still be green.

**Why human:** The probe-msvc step's override-mode code path (the `Write-Error + exit 1` branch for uninstalled toolset) can only be confirmed against the live runner. The code logic exists and is correct, but hard-fail behavior at step 1 preventing downstream steps from running requires a real dispatch. The 01-02-SUMMARY.md documents this was observed (available pins: 14.29, 14.44; job did not reach assert-msvc). This prior approval covers the negative case; flagged here for completeness.

---

## Gaps Summary

**TC-03 documentation gap:** The implementation diverged from the requirement text in a way that was intentional and well-documented in Plan 02 SUMMARY, but REQUIREMENTS.md was not updated. The requirement says "activated via `ilammy/msvc-dev-cmd@v1`" — ilammy was dropped because conda + ilammy both call `vcvarsall.bat`, overflowing CMD's 8192-char INCLUDE/LIB limit. Phase 1 preflight instead calls `cl.exe` by full path from `probe-msvc` outputs for _MSC_VER verification. No vcvarsall activation is needed for preflight. Phase 2 build will activate MSVC via cmake's VS generator.

**This is a REQUIREMENTS.md documentation gap, not a functional defect.** The preflight correctly identifies and validates the MSVC toolset. TC-04 (_MSC_VER assertion), TC-06 (nvcc assertion), and all other requirements are fully satisfied. TC-03's stated mechanism (ilammy) is absent, but its intent (MSVC toolset used correctly, validated by preflight) is preserved through a different mechanism. Recommend updating TC-03 text before Phase 2 to avoid confusion.

**Step count discrepancy:** Plan 03 SUMMARY claims 15 preflight steps but the file has 14. The SUMMARY's step 13 ("Activate MSVC via ilammy/msvc-dev-cmd") does not exist in the file (ilammy was dropped). Steps 14 and 15 in the SUMMARY correspond to steps 13 and 14 in the file (forensics summary and remediation hint). This is consistent with the ilammy removal — not a gap in the delivered functionality, but the SUMMARY step inventory is inaccurate.

---

## What is Working Well

- The phase goal is substantively achieved: a reliable preflight exists that fails loudly on toolchain drift and never silently accepts bad compilers
- The ban-grep invariant (TC-10) is correctly implemented with a self-match-safe regex character class approach
- All 5 preflight assertions have unique step IDs, strict-mode preludes, and named hard-fail paths with remediation hints
- The probe-msvc auto-select from CUDA compat matrix is more resilient than the original pin-based design (handles runner image rotation automatically)
- All commits documented in SUMMARY.md are verified present in git history (dfe478b, 09d7a78, 2cdf4c8, 23ce327, fa615aa, 642db9e, 0463a38)
- DOC-04 comment coverage at all 4 locked sites with correct paraphrase vocabulary (no literal banned flag in comments)

---

*Verified: 2026-04-16T18:00:00Z*
*Verifier: Claude (gsd-verifier)*
