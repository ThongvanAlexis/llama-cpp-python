---
phase: 4
slug: publish-consumer-ux
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-19
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

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

> Task IDs are placeholders (`04-01-NN`) — planner assigns final IDs. Each requirement below MUST appear in a plan task's `<automated>` verify block.

| Task ID     | Plan | Wave | Requirement | Test Type  | Automated Command | File Exists | Status |
|-------------|------|------|-------------|-----------|-------------------|-------------|--------|
| 04-01-NN    | 01   | 1    | PUB-01      | static    | `grep -cE '^  publish:' .github/workflows/build-wheels-cuda-windows.yaml` equals 1; then `awk '/^  publish:/,/^  [a-z]/' ... \| grep -cE 'runs-on:[[:space:]]*ubuntu-latest'` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-02      | static    | `grep -cE 'softprops/action-gh-release@v2' .github/workflows/build-wheels-cuda-windows.yaml` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-03      | static    | `awk '/^  publish:/,/^  [a-z]/' ... \| grep -cE 'needs:[[:space:]]*\[[[:space:]]*build[[:space:]]*,[[:space:]]*smoke-test[[:space:]]*\]'` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-04      | static    | `awk '/^  publish:/,/^  [a-z]/' ... \| grep -cE 'permissions:'` equals 1 AND `grep -cE '^      contents:[[:space:]]*write'` equals 1 within the publish block | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-05      | static    | `awk '/^  publish:/,/^  [a-z]/' ... \| grep -cE 'download-artifact@v4'` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-06      | static    | `grep -cE 'tag_name:[[:space:]]*v\$\{\{[[:space:]]*inputs\.release_version' .github/workflows/build-wheels-cuda-windows.yaml` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-07      | static    | `awk '/^  publish:/,/^  [a-z]/' ... \| grep -cE 'body:[[:space:]]*\|'` equals 1 AND body contains `needs.build.outputs.llama_sha` reference | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-08      | static    | `grep -cE 'files:[[:space:]]*' .github/workflows/build-wheels-cuda-windows.yaml` ≥ 1 within publish block; file pattern resolves `*.whl` | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-09      | static    | `grep -cE 'draft:[[:space:]]*false' .github/workflows/build-wheels-cuda-windows.yaml` equals 1; `grep -cE 'prerelease:[[:space:]]*false'` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | PUB-10      | static    | `grep -cE 'fail_on_unmatched_files:[[:space:]]*true' .github/workflows/build-wheels-cuda-windows.yaml` equals 1 | ✅ | ⬜ pending |
| 04-01-NN    | 01   | 1    | DOC-01      | static    | `grep -cE '^## Install.*[Ww]indows.*CUDA' README.md` equals 1 AND README contains `https://github.com/ThongvanAlexis/llama-cpp-python/releases` reference | ❌ (W0) | ⬜ pending |
| 04-01-NN    | 01   | 1    | DOC-02      | static    | `grep -cE '(nvidia-smi\|561\.17\|driver.*\b5[0-9][0-9]\b)' README.md` ≥ 1 | ❌ (W0) | ⬜ pending |
| 04-01-NN    | 01   | 1    | DOC-03      | static    | `grep -cE 'pip install.*--extra-index-url.*github\.com/ThongvanAlexis/llama-cpp-python/releases' README.md` ≥ 1 OR canonical install snippet present | ❌ (W0) | ⬜ pending |
| 04-01-NN    | 01   | 1    | DOC-05      | dispatch  | After green publish dispatch: `gh release view v<ver>-cu126-win --json body -q .body \| grep -cE 'llama\.cpp.*(SHA\|sha)'` ≥ 1 | — (runtime) | ⬜ pending |
| 04-01-NN    | 01   | 1    | Gate assert | static    | lint-workflow grep step: `grep -cE 'needs:[[:space:]]*\[[[:space:]]*build[[:space:]]*,[[:space:]]*smoke-test[[:space:]]*\]' build-wheels-cuda-windows.yaml` equals 1; deliberate removal must surface `::error::` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `README.upstream.md` — Wave 0 creates via `git mv README.md README.upstream.md`; downstream tasks rewrite a fresh fork-focused `README.md`.
- [ ] New `README.md` skeleton — headings for "Install (Windows CUDA)", "Driver requirements", "Canonical install command" must exist before DOC-grep invariants pass.
- [ ] Build job `outputs:` block — verify with `grep -nE '^    outputs:' .github/workflows/build-wheels-cuda-windows.yaml` under the `build:` job; add `llama_sha` + `selected_toolset` outputs if absent so `needs.build.outputs.*` references resolve.
- [ ] `actionlint` — already bootstrapped ad-hoc inside the existing `lint-workflow` job; no global install step required.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| End-to-end `pip install` from the published Release URL works in a fresh Windows venv with CUDA 12.6 runtime | PUB-02, DOC-03 | Requires real GitHub.com Release asset + real Windows host; cannot run inside CI without circular dependency on the thing being validated | After a green publish dispatch: on a fresh Windows dev box, `py -3.11 -m venv fresh && fresh\Scripts\activate && pip install llama-cpp-python --index-url https://github.com/ThongvanAlexis/llama-cpp-python/releases/download/v<ver>-cu126-win/` — import must succeed and `from llama_cpp import Llama` must return without DLL-load errors. |
| Release body renders correctly on github.com (markdown tables, code fences) | PUB-07, DOC-05 | Requires visual inspection of rendered markdown | Open `https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v<ver>-cu126-win` in a browser; confirm forensics table renders, llama.cpp SHA is present and links to the pinned commit, install snippet is syntax-highlighted. |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references (README.md rewrite, upstream preservation, build job outputs block)
- [ ] No watch-mode flags
- [ ] Feedback latency < 10 s for static checks; one full dispatch for end-to-end gate
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
