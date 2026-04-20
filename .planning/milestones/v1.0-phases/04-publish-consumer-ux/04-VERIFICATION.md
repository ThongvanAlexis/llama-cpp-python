---
phase: 04-publish-consumer-ux
verified: 2026-04-19T00:00:00Z
status: passed
score: 5/5 must-haves verified
gaps: []
human_verification:
  - test: "Install the wheel on a fresh Windows host with only an NVIDIA driver (no CUDA Toolkit)"
    expected: "`pip install path/to/wheel.whl` succeeds; `from llama_cpp import Llama; print('ok')` exits 0; no DLL-load error dialog"
    why_human: "Requires a physical Windows + GPU test environment; no automated substitute. CUDA-bundling disclosure was patched based on user visual inspection, but the installation flow has not been executed on a clean host."
---

# Phase 4: Publish & Consumer UX Verification Report

**Phase Goal:** On green smoke-test, the publish job uploads the built wheel as a GitHub Release asset via `softprops/action-gh-release@v2` at tag `v<base_version>-cu126-win`, with a forensics release body embedding the 12-char llama.cpp submodule SHA; the repo README is overwritten to a fork-focused document explaining manual download + `pip install path/to/wheel.whl`; `REQUIREMENTS.md` and `PROJECT.md` are amended to record the 2026-04-19 scope reduction.
**Verified:** 2026-04-19
**Status:** PASSED (all 5 active requirements verified; 9 out-of-scope IDs correctly parked)
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `publish:` job exists on `ubuntu-latest` with `needs: [build, smoke-test]`, job-level `permissions: contents: write`, using `actions/download-artifact@v4` (name: `cuda-wheel`) + `softprops/action-gh-release@v2` with `target_commitish: ${{ github.sha }}` + `fail_on_unmatched_files: true` | VERIFIED | Workflow line 1702-1820; grep confirms each required clause at lines 1703, 1704, 1707, 1716, 1812, 1819, 1820 |
| 2 | `build:` job has top-level `outputs:` block surfacing `llama_sha`, `selected_toolset`, `selected_full_version` | VERIFIED | Workflow lines 126-129; all three entries point to correct step outputs (`steps.assert-submodule.outputs.llama_sha`, `steps.probe-msvc.outputs.selected_toolset`, `steps.probe-msvc.outputs.selected_full_version`) |
| 3 | `lint-workflow` job contains a character-class-guarded presence-grep for `needs: [build, smoke-test]` on the publish job (count=1, anchored `^    needs[:]`) | VERIFIED | Workflow line 1669 `'Lint: assert publish-gate on smoke-test'`; line 1679 anchored grep pattern; `04-02-run-meta.json` shows `lint-workflow` conclusion=success |
| 4 | `README.md` is a fork-focused document with `## Install (Windows CUDA)` section, NVIDIA driver floor (≥ 561.17), manual download + `pip install path/to/wheel.whl` flow, fork URL, Attribution to `README.upstream.md`; upstream README preserved byte-for-byte at `README.upstream.md` | VERIFIED | README.md exists at ~99 lines with all required H2s; `README.upstream.md` = 824 lines; neither contains `--extra-index-url` |
| 5 | End-to-end `workflow_dispatch` created GitHub Release at `v0.3.20-cu126-win` with wheel as asset, SHA-256 matching publish job log, release body containing 12-char llama.cpp SHA and forensics table | VERIFIED | `04-02-release.json`: tagName=`v0.3.20-cu126-win`, asset SHA-256=`47649ad0...adbeed8`, body contains `227ed28e128e` (12 chars) in narrative + forensics table; `04-02-run-meta.json`: all 4 jobs conclusion=success |

**Score: 5/5 truths verified**

---

## Required Artifacts

| Artifact | Expected | Status | Evidence |
|----------|----------|--------|----------|
| `.github/workflows/build-wheels-cuda-windows.yaml` | publish job + lint-gate + build outputs block | VERIFIED | File exists; 1842 lines; publish job at line 1702; build outputs at line 126; lint gate at line 1669 |
| `README.md` | Fork-focused consumer install documentation | VERIFIED | File exists, 99 lines, all 4 required H2 sections, `561.17`, releases URL, `pip install`, attribution to `README.upstream.md` |
| `README.upstream.md` | Preserved upstream README (824 lines, byte-for-byte) | VERIFIED | File exists; `wc -l` = 824 |
| `.planning/REQUIREMENTS.md` | Scope-reduction amendments; PUB-03..10 + DOC-03 in Out of Scope; DOC-01 rewritten | VERIFIED | DOC-01 line 65: `[x]` with "manual download" text; Out of Scope table at lines 104-105 contains both PUB-03..10 and DOC-03 rows; traceability rows updated at lines 163-173 |
| `.planning/PROJECT.md` | Core Value updated + Key Decision row dated 2026-04-19 | VERIFIED | Core Value at line 9 reads manual-download wording; Key Decisions table at line 97 contains 2026-04-19 scope reduction entry; last updated footer at line 100 |
| `04-02-run-meta.json` | Dispatch forensics: conclusion=success, all jobs green | VERIFIED | File exists; conclusion=`success`; publish job conclusion=`success`; lint-workflow conclusion=`success`; build+smoke-test=`success` |
| `04-02-release.json` | Live Release JSON: correct tagName, asset, body with 12-char SHA | VERIFIED | tagName=`v0.3.20-cu126-win`; asset name matches Phase 2 regex; body contains `227ed28e128e` (12 hex chars); dispatch run URL embedded |
| `04-02-publish-log.txt` | Publish job step log: wheel download, metadata parse, body assembly, softprops upload | VERIFIED | File exists; log confirms `name: cuda-wheel` download, nullglob array for wheel detection, 12-char SHA parsed, softprops invoked with `tag_name: v0.3.20-cu126-win` |

---

## Key Link Verification

| From | To | Via | Status | Evidence |
|------|----|-----|--------|----------|
| publish job metadata step | 12-char llama.cpp SHA (`llama_sha_12`) | `sed -E 's/^.*\+cu[0-9]+\.ll([a-f0-9]{12})-.*/\1/'` on wheel filename | VERIFIED | Workflow line 1744; publish log shows `LLAMA_SHA_12=$(echo "$WHEEL_NAME" | sed ...)` producing `227ed28e128e`; release body confirms 12-char substitution |
| publish job | `needs.build.outputs.selected_toolset` | job-level outputs on `build:` surfacing `steps.probe-msvc.outputs.selected_toolset` | VERIFIED | Workflow line 128 (build outputs), line 1800 (forensics table row `MSVC toolset`); live release body shows `14.44` matching actual runner value |
| publish job | `cuda-wheel` artifact | `actions/download-artifact@v4` with `name: cuda-wheel` | VERIFIED | Workflow line 1716; publish log line 34 confirms `name: cuda-wheel`; artifact ID 6517113259 downloaded |
| publish job softprops upload | GitHub Release at `v0.3.20-cu126-win` | `softprops/action-gh-release@v2` with `target_commitish: ${{ github.sha }}` | VERIFIED | `04-02-release.json`: `targetCommitish=973aeff4441c38ac7e9c78e6db15518623b34dc1`; asset SHA-256 matches publish log `wheel_sha256` (byte-identical upload) |
| lint-workflow job | publish job's `needs: [build, smoke-test]` | anchored grep `^    needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]` (count=1) | VERIFIED | Workflow line 1679; grep of workflow at `^    needs[:]` returns exactly 1 match at line 1703; lint-workflow conclusion=success in run 24623672816 |
| README Install section | GitHub Releases page for ThongvanAlexis/llama-cpp-python | hardcoded URL | VERIFIED | README.md line 39: `https://github.com/ThongvanAlexis/llama-cpp-python/releases` |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| PUB-01 | 04-01-PLAN.md | `publish` job runs on `ubuntu-latest` | SATISFIED | Workflow line 1704: `runs-on: ubuntu-latest`; `04-02-run-meta.json` job `publish` conclusion=success on ubuntu runner (confirmed by publish log showing `Ubuntu 24.04.4`) |
| PUB-02 | 04-01-PLAN.md, 04-02-PLAN.md | Wheel uploaded as GitHub Release asset via `softprops/action-gh-release@v2` | SATISFIED | Workflow line 1812: `uses: softprops/action-gh-release@v2`; live Release at `v0.3.20-cu126-win` with wheel asset `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` (356,871,830 bytes) |
| DOC-01 | 04-01-PLAN.md | README `Install (Windows CUDA)` section with manual download + `pip install path/to/wheel.whl`, releases URL, driver floor | SATISFIED | README.md lines 18-57: `## Install (Windows CUDA)` present; `561.17` driver floor at line 24; releases URL at line 39; `pip install path\to\...` at line 47 |
| DOC-02 | 04-01-PLAN.md | README notes NVIDIA driver minimum (≥ 560.x / ≥ 561.17) | SATISFIED | README.md lines 24, 31, 33: `≥ 561.17` (3 occurrences) |
| DOC-05 | 04-01-PLAN.md, 04-02-PLAN.md | Release notes record llama.cpp submodule SHA at build time | SATISFIED | `04-02-release.json` body: narrative line `"built 2026-04-19 against llama.cpp @ \`227ed28e128e\`"`; forensics table row `llama.cpp SHA | \`227ed28e128e\``; 12-char hex (not 7-char) confirmed; `steps.meta.outputs.llama_sha_12` substitution verified via publish log |
| PUB-03 | Out of Scope (2026-04-19) | PEP 503 wheel path placement on gh-pages | OUT OF SCOPE — correctly parked | REQUIREMENTS.md Out of Scope table line 104; Traceability line 163: `Deferred to v2 / Out of Scope` |
| PUB-04 | Out of Scope (2026-04-19) | `index.html` regeneration | OUT OF SCOPE — correctly parked | Same consolidated row as PUB-03 in Out of Scope table |
| PUB-05 | Out of Scope (2026-04-19) | PEP 503 normalized name | OUT OF SCOPE — correctly parked | Same row |
| PUB-06 | Out of Scope (2026-04-19) | `#sha256=` URL fragments | OUT OF SCOPE — correctly parked | Same row |
| PUB-07 | Out of Scope (2026-04-19) | `data-requires-python` attrs | OUT OF SCOPE — correctly parked | Same row |
| PUB-08 | Out of Scope (2026-04-19) | `keep_files: true` | OUT OF SCOPE — correctly parked | Same row |
| PUB-09 | Out of Scope (2026-04-19) | gh-pages concurrency block | OUT OF SCOPE — correctly parked | Same row |
| PUB-10 | Out of Scope (2026-04-19) | Post-publish Fastly probe | OUT OF SCOPE — correctly parked | Same row |
| DOC-03 | Out of Scope (2026-04-19) | Fastly cache delay note | OUT OF SCOPE — correctly parked | REQUIREMENTS.md Out of Scope table line 105; Traceability line 173: `Deferred to v2 / Out of Scope` |

**Active requirements satisfied: 5/5**
**Out-of-scope IDs correctly parked: 9/9 (PUB-03..10, DOC-03)**

---

## Anti-Requirement Verification

| Anti-requirement | Check | Status |
|-----------------|-------|--------|
| NO `peaceiris/actions-gh-pages` anywhere in workflow | `grep -E 'peaceiris/actions-gh-pages'` | CLEAN — zero matches |
| NO `gh-pages-publish` concurrency block | `grep -E 'gh-pages-publish'` | CLEAN — zero matches |
| NO `--extra-index-url` in README or workflow | Grepped both files | CLEAN — zero matches in both `README.md` and workflow |
| NO post-publish HTTP / Fastly probe step | `grep -cE 'curl.*fastly|curl.*github\.io'` on publish-log.txt | CLEAN — no such steps in publish job log |
| Publish job wall time under 10 min | `04-02-run-meta.json`: publish started 09:20:28Z, completed 09:21:05Z | CLEAN — 37 seconds actual (10-minute timeout bound never approached) |
| NO `actions/checkout@v4` inside publish job | Grepped publish job body | CLEAN — publish job has no checkout step; all inputs from download-artifact + `needs.build.outputs.*` |
| NO workflow-level `permissions: contents: write` escalation | Workflow line 119-120: `permissions: contents: read` | CLEAN — publish job declares job-level `permissions: contents: write` only; workflow-level stays `read` |

---

## Notable Deviations (All Remediated Before Close)

Four deviations occurred during phase execution. All were auto-fixed before the phase was declared complete.

**Deviation 1 — Gate-lint anchor (Plan 04-01, Task 2)**
The plan's lint pattern `needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]` (unanchored) matched 6 lines in the workflow instead of 1, because comments, error-message echo strings, and release-body template cells all contained the literal phrase. Fixed by prefixing the pattern with `^    ` (4-space YAML job-declaration indent), making the count exactly 1. Error/log strings were simultaneously paraphrased to use "colon"/"dash" so they remain regex-invisible.

**Deviation 2 — README `--extra-index-url` wording (Plan 04-01, Task 3)**
The plan's README prose included `no \`--extra-index-url\` index` in the intro paragraph. The anti-requirement grep (`! grep -E 'extra-index-url' README.md`) triggered on this informative sentence. Fixed by rephrasing to `no PEP 503 pip index on GitHub Pages` — same consumer-facing meaning, no literal flag string.

**Deviation 3 — SC2012 nullglob array (Plan 04-02, pre-dispatch)**
The publish job's metadata step originally used `ls dist/*.whl | head -1` for wheel detection. The Phase 1 `actionlint` integration (SC2012 — "useless use of `ls`") flagged this and failed the lint-workflow job on first dispatch, blocking the entire run. Fixed by replacing with `shopt -s nullglob; files=(dist/*.whl); WHEEL="${files[0]:-}"`. Re-dispatched from commit `973aeff` (the fix), which became the release's `targetCommitish`.

**Deviation 4 — CUDA Toolkit disclosure gap (Plan 04-02, visual checkpoint)**
The user's visual inspection of the live release page revealed that the release body did not disclose that the wheel bundles the CUDA 12.6 runtime DLLs (consumers might incorrectly infer they need to install CUDA Toolkit 12.6.3 locally). Fixed via two-path remediation: `gh release edit` on the live `v0.3.20-cu126-win` body (consumers see it immediately) + workflow template update (next dispatch inherits the disclosure natively). The disclosure now appears at position 2 in the body, before the Install numbered steps.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none) | — | — | — | — |

No TODO/FIXME/placeholder comments, empty implementations, or console.log-only stubs found in phase deliverables.

---

## Human Verification Required

### 1. Fresh-host wheel install smoke test

**Test:** On a Windows 10/11 host with an NVIDIA GPU (driver ≥ 561.17) and NO CUDA Toolkit installed, download `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` from `https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v0.3.20-cu126-win` and run:
```
pip install path\to\llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl
python -c "from llama_cpp import Llama; print('ok')"
```
**Expected:** `pip install` exits 0; `python` prints `ok` with no DLL-load error, no Access Violation dialog, no segfault.
**Why human:** Requires a physical Windows + GPU environment. The CI smoke-test job (Phase 3) runs on a driverless `windows-2022` runner and uses a CUDA-architecture stub (`cuobjdump`) as a proxy rather than actual GPU inference. The CUDA-runtime-bundling disclosure (Deviation 4 above) was added precisely because this host-install path had not been exercised; a human test on a driver-only machine would close the loop on DOC-01's consumer-facing claim.

---

## Gaps Summary

No gaps. All 5 active Phase 4 requirements (PUB-01, PUB-02, DOC-01, DOC-02, DOC-05) are satisfied with concrete evidence at the workflow-code level and dispatch-forensics level. All 9 deferred IDs (PUB-03..10, DOC-03) are correctly recorded as Out of Scope in REQUIREMENTS.md with a consolidated rationale row and traceability entries flipped to `Deferred to v2 / Out of Scope`. The 4 deviations encountered during execution were each remediated before the phase was closed.

The single human-verification item above is a forward-looking confidence test on DOC-01's consumer-install claim, not a gap in the phase deliverables.

---

*Verified: 2026-04-19*
*Verifier: Claude (gsd-verifier)*
