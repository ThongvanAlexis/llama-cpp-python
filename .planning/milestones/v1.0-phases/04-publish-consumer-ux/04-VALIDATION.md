---
phase: 4
slug: publish-consumer-ux
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-19
approved: 2026-04-19
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.
>
> **Amended 2026-04-19:** The 2026-04-19 CONTEXT.md scope reduction moved **PUB-03..PUB-10** and **DOC-03** to the "Out of Scope" section of REQUIREMENTS.md (see Plan 04-01 Task 3). The rows for those requirements below have been rewritten from workflow-static-grep verification to **amendment verification** — we no longer verify workflow YAML properties for the deferred requirements; we verify that the scope-reduction amendment landed correctly in REQUIREMENTS.md.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | GitHub Actions self-verification — actionlint + shell-level grep invariants. No pytest (Phase 4 adds zero Python code). |
| **Config file** | `.github/workflows/build-wheels-cuda-windows.yaml` (existing) |
| **Quick run command** | `bash <(curl -sSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash) && ./actionlint -color .github/workflows/build-wheels-cuda-windows.yaml` |
| **Full suite command** | `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.6.3 && gh run watch` |
| **Estimated runtime** | Quick: ~5 s (actionlint + grep). Full dispatch: ~2–3 h on warm caches (publish requires prior build+smoke-test artifacts). |

---

## Sampling Rate

- **After every task commit:** Run `actionlint .github/workflows/build-wheels-cuda-windows.yaml` + task-specific grep invariants listed below.
- **After every plan wave:** Run the quick command plus the full-file `actionlint` pass.
- **Before `/gsd:verify-work`:** One full `workflow_dispatch` must produce a green publish job, a real GitHub Release at the expected tag, an uploaded wheel asset, and a release body containing the llama.cpp submodule SHA + forensics table.
- **Max feedback latency:** ~5 s for static checks; single human-driven dispatch for the end-to-end gate.

---

## Per-Task Verification Map

> Task IDs are `04-{plan}-T{task}`. Each requirement below MUST appear in a plan task's `<automated>` verify block OR be verified via the amendment-style row below (for the 2026-04-19 scope-reduced requirements).

| Task ID     | Plan | Wave | Requirement | Test Type   | Automated Command | File Exists | Status |
|-------------|------|------|-------------|-------------|-------------------|-------------|--------|
| 04-01-T2    | 01   | 1    | PUB-01      | static      | `grep -cE '^  publish:' .github/workflows/build-wheels-cuda-windows.yaml` equals 1; then `awk '/^  publish:/,/^  [a-z][a-z-]*:/' ... \| grep -cE 'runs-on:\s*ubuntu-latest'` equals 1 | ✅ | ⬜ pending |
| 04-01-T2    | 01   | 1    | PUB-02      | static      | `grep -cE 'softprops/action-gh-release@v2' .github/workflows/build-wheels-cuda-windows.yaml` equals 1 | ✅ | ⬜ pending |
| 04-01-T3    | 01   | 1    | PUB-03      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-04      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03\\.\\.PUB-10\\\|PUB-04'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-05      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03\\.\\.PUB-10\\\|PUB-05'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-06      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03\\.\\.PUB-10\\\|PUB-06'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-07      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03\\.\\.PUB-10\\\|PUB-07'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-08      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03\\.\\.PUB-10\\\|PUB-08'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-09      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-03\\.\\.PUB-10\\\|PUB-09'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | PUB-10      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'PUB-10'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-01-T3    | 01   | 1    | DOC-01      | static      | `grep -cE '^- \[ \] \*\*DOC-01\*\*.*(manual download\|pip install path)' .planning/REQUIREMENTS.md` equals 1 AND `grep -cE '^## Install \(Windows CUDA\)' README.md` equals 1 AND README contains `https://github.com/ThongvanAlexis/llama-cpp-python/releases` reference | ✅ | ⬜ pending |
| 04-01-T3    | 01   | 1    | DOC-02      | static      | `grep -cE '(nvidia-smi\|561\\.17\|driver.*\\b5[0-9][0-9]\\b)' README.md` ≥ 1 | ✅ | ⬜ pending |
| 04-01-T3    | 01   | 1    | DOC-03      | amendment   | `awk '/^## Out of Scope/,/^## Traceability/' .planning/REQUIREMENTS.md \| grep -q 'DOC-03'` | ✅ | ⬜ pending · Out of Scope (amended via plan 04-01 Task 3, 2026-04-19) |
| 04-02-T1    | 02   | 2    | DOC-05      | dispatch    | After green publish dispatch: `jq -r '.body' .planning/phases/04-publish-consumer-ux/04-02-release.json \| grep -qE '\`[a-f0-9]{12}\`'` (12-char SHA in body, rendered as inline code) | — (runtime) | ⬜ pending |
| 04-01-T2    | 01   | 1    | Gate assert | static      | lint-workflow grep step: `grep -cE 'needs:\s*\[\s*build\s*,\s*smoke-test\s*\]' build-wheels-cuda-windows.yaml` equals 1; deliberate removal must surface `::error::` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky · ⊘ Out of Scope (scope reduction amended)*

---

## Wave 0 Requirements

- [x] `README.upstream.md` — Wave 0 creates via `git mv README.md README.upstream.md`; downstream tasks rewrite a fresh fork-focused `README.md`. *(owned by plan 04-01 Task 1)*
- [x] New `README.md` skeleton — headings for "Install (Windows CUDA)", "How the wheel is built", "Attribution", "License" must exist before DOC-grep invariants pass. *(owned by plan 04-01 Task 1)*
- [x] Build job `outputs:` block — verify with `grep -nE '^    outputs:' .github/workflows/build-wheels-cuda-windows.yaml` under the `build:` job; add `llama_sha` (7-char, diagnostic) + `selected_toolset` + `selected_full_version` outputs so `needs.build.outputs.*` references resolve. The 12-char SHA needed by DOC-05 is parsed from the wheel filename in Plan 04-01 Task 2's metadata step (`steps.meta.outputs.llama_sha_12`) — no new `assert-submodule` output is required.
- [x] `actionlint` — already bootstrapped ad-hoc inside the existing `lint-workflow` job; no global install step required.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| End-to-end `pip install path/to/wheel.whl` from the downloaded Release asset works in a fresh Windows venv with CUDA 12.6 runtime | PUB-02, DOC-01 | Requires real GitHub.com Release asset + real Windows host; cannot run inside CI without circular dependency on the thing being validated | After a green publish dispatch: on a fresh Windows dev box, download the `.whl` from `https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v<ver>-cu126-win`, then `py -3.11 -m venv fresh && fresh\Scripts\activate && pip install path\to\llama_cpp_python-<ver>+cu126.ll<sha>-cp311-cp311-win_amd64.whl` — import must succeed and `from llama_cpp import Llama` must return without DLL-load errors. |
| Release body renders correctly on github.com (markdown tables, code fences, 12-char SHA as inline code) | DOC-05 | Requires visual inspection of rendered markdown | Open `https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v<ver>-cu126-win` in a browser; confirm forensics table renders, llama.cpp SHA (12-char) is present in the first narrative line AND in the forensics table row, and the `<sha12>` in both locations matches the `<sha12>` in the attached wheel's filename. |

---

## Scope Reduction — 2026-04-19

The following requirements were moved to REQUIREMENTS.md "Out of Scope" by plan 04-01 Task 3 on 2026-04-19. Their verification in this document is **amendment-style** (verify the deferral landed in REQUIREMENTS.md) rather than **static** (verify workflow YAML properties). See CONTEXT.md for the rationale.

| ID | Original intent | New verification |
|----|-----------------|------------------|
| PUB-03 | gh-pages PEP 503 pip index path placement | Out of Scope — grep REQUIREMENTS.md "Out of Scope" section for `PUB-03` |
| PUB-04 | index.html regeneration from `get-releases.sh` | Out of Scope — part of consolidated `PUB-03..PUB-10` row |
| PUB-05 | PEP 503 normalized name | Out of Scope — part of consolidated `PUB-03..PUB-10` row |
| PUB-06 | `#sha256=` URL fragment | Out of Scope — part of consolidated `PUB-03..PUB-10` row |
| PUB-07 | `data-requires-python` attribute | Out of Scope — part of consolidated `PUB-03..PUB-10` row |
| PUB-08 | `peaceiris/actions-gh-pages@v4` with `keep_files: true` | Out of Scope — part of consolidated `PUB-03..PUB-10` row |
| PUB-09 | `concurrency: gh-pages-publish` block | Out of Scope — part of consolidated `PUB-03..PUB-10` row |
| PUB-10 | Post-publish Fastly HTTP probe | Out of Scope — grep REQUIREMENTS.md "Out of Scope" section for `PUB-10` |
| DOC-03 | Fastly cache delay note | Out of Scope — grep REQUIREMENTS.md "Out of Scope" section for `DOC-03` |

DOC-01's verification remains **static** but the grep was rewritten to target the amended bullet wording: `grep -cE '^- \[ \] \*\*DOC-01\*\*.*(manual download|pip install path)' .planning/REQUIREMENTS.md` equals 1. This matches CONTEXT.md's DOC-01 reinterpretation (manual download + `pip install path/to/wheel.whl`, not `--extra-index-url`).

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (README.md rewrite, upstream preservation, build job outputs block)
- [x] No watch-mode flags
- [x] Feedback latency < 10 s for static checks; one full dispatch for end-to-end gate
- [x] `nyquist_compliant: true` set in frontmatter
- [x] Scope-reduction amendments (PUB-03..10 + DOC-03) verified via REQUIREMENTS.md "Out of Scope" greps, not workflow statics

**Approval:** approved 2026-04-19
</content>
</invoke>