---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: verifying
stopped_at: "Completed 01-03-PLAN.md — Phase 1 complete (3/3 plans). SUMMARY written, STATE updated. Next: /gsd:verify-work for Phase 1, then Phase 2 planning."
last_updated: "2026-04-16T12:59:11.081Z"
last_activity: 2026-04-16 — Phase 1 complete. Plan 03 added forensics summary (11-property fingerprint table in Actions UI Summary tab) + DOC-04 inline comments at 4 locked sites. All 16 Phase 1 requirements verified (WF-01..05, TC-01..10, DOC-04). Workflow file feature-complete.
progress:
  total_phases: 4
  completed_phases: 1
  total_plans: 3
  completed_plans: 3
  percent: 100
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-15)

**Core value:** Produce a Windows x64 CUDA wheel that actually works at runtime (loads a model, runs inference without segfault) and is installable via `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu126`.
**Current focus:** Phase 1 COMPLETE. Ready for /gsd:verify-work, then Phase 2 (Build & Cache).

## Current Position

Phase: 1 of 4 (Scaffold & Toolchain Pinning) -- COMPLETE
Plan: 3 of 3 in current phase (all plans complete)
Status: Phase 1 complete, awaiting verification
Last activity: 2026-04-16 — Phase 1 complete. Plan 03 added forensics summary (11-property fingerprint table in Actions UI Summary tab) + DOC-04 inline comments at 4 locked sites. All 16 Phase 1 requirements verified (WF-01..05, TC-01..10, DOC-04). Workflow file feature-complete.

Progress: [██████████] 100% (Phase 1)

## Performance Metrics

**Velocity:**
- Total plans completed: 3
- Average duration: ~9.3h (skewed by Plan 02's 6-dispatch empirical discovery cycle)
- Total execution time: ~28.1 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. Scaffold & Toolchain Pinning | 3/3 | ~28.1h | ~9.3h |
| 2. Build & Cache | 0 | — | — |
| 3. Smoke Test (Publish Gate) | 0 | — | — |
| 4. Publish & Consumer UX | 0 | — | — |

**Recent Trend:**
- Last 5 plans: P01 (2 min), P02 (~24h multi-session), P03 (~4h)
- Trend: Plan 02 was unusually long due to empirical discovery of runner constraints (6 dispatches). Plan 03 was faster (forensics + docs with 1 dispatch iteration). Phase 1 complete.

*Updated after each plan completion*
| Phase 01-scaffold-toolchain-pinning P01 | 2 min | 1 tasks | 1 files |
| Phase 01-scaffold-toolchain-pinning P02 | ~24h | 3 tasks | 1 files |
| Phase 01-scaffold-toolchain-pinning P03 | ~4h | 3 tasks | 1 files |

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work (from PROJECT.md + research):

- [Phase 1]: New workflow file `build-wheels-cuda-windows.yaml`, not edit of upstream's `build-wheels-cuda.yaml` (merge-conflict hygiene)
- [Phase 1]: MSVC auto-selected from runner (not pinned). `ilammy/msvc-dev-cmd` dropped — conda + ilammy both call vcvarsall.bat, overflowing CMD's 8192-char INCLUDE/LIB limit. Phase 1 calls cl.exe by full path for _MSC_VER check. **Phase 2 must NOT use ilammy** — use cmake's VS generator or a clean vcvarsall subprocess instead
- [Phase 1]: `-allow-unsupported-compiler` banned (historical runtime-segfault trap, upstream #1543); enforced by grep-assert
- [Phase 2]: save caches on failure (`if: always()`) — most dev iterations will be failing builds
- [Phase 3]: Smoke test is a hard publish gate (`needs: [build, smoke-test]`), not an advisory step
- [Phase 4]: Publish via `peaceiris/actions-gh-pages@v4` with `keep_files: true` + `concurrency: group: gh-pages-publish, cancel-in-progress: false`
- [Phase 01-scaffold-toolchain-pinning]: actionlint installed via official curl one-liner (not a wrapping action) — locked for Plan 03
- [Phase 01-scaffold-toolchain-pinning]: Ban-grep diagnostic messages use paraphrases ('banned CMake flag', 'compiler-bypass workaround flag') — preserves count-equals-1 invariant for VALIDATION quick command; same policy applies to Plan 03 Task 2 DOC-04 inline comments
- [Phase 01-scaffold-toolchain-pinning]: Strict bash prelude 'set -euo pipefail' adopted as Plan 02/03 reusable idiom for all shell:bash blocks
- [Phase 01-scaffold-toolchain-pinning 2026-04-15]: OQ1 resolved — `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` component NOT available on current windows-2022 image (actions/runner-images#9701). Chose Option B: bump default toolchain from CUDA 12.4.1 + MSVC 14.39 to CUDA 12.6.3 + MSVC 14.40. Workflow also gains a toolset→VS-suffix hashtable lookup (14.39/17.9, 14.40/17.10, 14.41/17.11, 14.42/17.12) with throw-on-unknown; ilammy `vs-version` now derives dynamically from the lookup (no more hardcoded `[17.9,17.10)`). Pre-dispatch sanity: `12.6.3` exists on `nvidia/label/cuda-12.6.3` noarch channel.
- [Phase 01-scaffold-toolchain-pinning 2026-04-15]: Discovered + fixed ban-grep self-match bug in Plan 01 lint step: `grep -nH 'allow-unsupported-compiler'` matched its own source line causing every dispatch to fail lint. Replaced with `grep -nHE 'allow[-]unsupported[-]compiler'` — regex character class `[-]` contains only `-`, matches real hyphens in targets but not the pattern's own literal text. Invariant flips from count=1 to count=0.
- [Phase 01-scaffold-toolchain-pinning 2026-04-16]: MSVC probe step refactored from install-on-missing to enumerate-from-disk. The vs_installer approach failed for both 14.39 AND 14.40 because Microsoft retired those VC components from the windows-2022 channel manifest (actions/runner-images#9701). New approach: enumerate `VC\Tools\MSVC\*` directories on disk (instant, ~0ms); if requested pin present, use it; if not, fail with full list of available pins. Benefits: (1) no dependency on retired channel components, (2) every dispatch log is self-documenting, (3) ilammy simplified to `toolset:` input (no vs-version range derivation). Removed: toolsetToVs hashtable, vs_installer code, vs_version_range/component_id outputs.
- [Phase 01-scaffold-toolchain-pinning 2026-04-16]: Corrected CUDA _MSC_VER cap model. Original assumption: each CUDA minor version only supports one more MSVC minor (per-minor caps). Actual (from NVIDIA host_config.h): caps are per-generation — CUDA 12.2-12.3 cap at 1939 (`_MSC_VER >= 1940`), CUDA 12.4-12.9 cap at 1949 (`_MSC_VER >= 1950`). This means MSVC 14.44 (_MSC_VER=1944) works with ANY CUDA 12.4+. Sources: nerfstudio#3157, NVIDIA forums, Microsoft _MSC_VER docs. CUDA 12.7 was never released (NVIDIA skipped to 12.8).
- [Phase 01-scaffold-toolchain-pinning 2026-04-16]: MSVC auto-select design adopted. msvc_toolset input defaults to 'auto' (was '14.40'). Probe step now: (1) looks up CUDA major.minor in a compat matrix to get _MSC_VER cap, (2) enumerates installed toolsets from disk, (3) picks newest compatible toolset automatically. Override mode (explicit pin) validates both installed AND compatible. Benefits: resilient to runner image rotation (adapts to whatever MSVC is available), eliminates need to track which specific MSVC version is on the runner. 3rd dispatch found 14.29 and 14.44 on runner; auto-select will pick 14.44 for CUDA 12.6 (1944 <= 1949).

- [Phase 01-scaffold-toolchain-pinning 2026-04-16]: Plan 03 forensics summary uses `$env:CONDA_PREFIX\Library` for CUDA path (not `$env:CUDA_PATH`) because mamba CUDA install sets CONDA_PREFIX but not CUDA_PATH. Phase 2 should use the same pattern for any CUDA path references.
- [Phase 01-scaffold-toolchain-pinning 2026-04-16]: Phase 1 COMPLETE. All 16 requirements verified (WF-01..05, TC-01..10, DOC-04). Workflow file has 15 preflight steps + 2-step lint-workflow job. Forensics summary renders 11-property table in Actions UI Summary tab on green; remediation hint block on red. DOC-04 comments at 4 locked sites with ban-grep-safe paraphrases.

### Pending Todos

[From .planning/todos/pending/ — ideas captured during sessions]

None yet.

### Blockers/Concerns

[Issues that affect future work]

- [Phase 1 resolved 2026-04-16]: MSVC component availability on windows-2022 — FULLY resolved. Both 14.39 and 14.40 components were unavailable via vs_installer channel manifest. Root cause: Microsoft retires older VC components from the manifest but the toolset files remain on disk. Solution: enumerate `VC\Tools\MSVC\*` directories instead of querying channel manifest (commit fb1497d). Next dispatch will reveal which pins are actually installed; default will be updated to match.
- [Phase 2 pre-plan risk]: `hashFiles('vendor/llama.cpp/.git/HEAD')` behaves differently when submodule `.git` is a file (git-dir indirection) vs a directory — verify empirically on first Phase 1 run.
- [Phase 2 CRITICAL]: **Do NOT use `ilammy/msvc-dev-cmd`** for VS environment activation. Conda's activation hook calls vcvarsall.bat internally, and a second vcvars call (from ilammy) overflows CMD's 8192-char INCLUDE/LIB limit. Options for Phase 2 build step: (a) cmake's "Visual Studio 17 2022" generator handles VS activation natively, (b) call vcvarsall.bat in a clean pwsh subprocess before the build, (c) set INCLUDE/LIB/PATH manually from probe-msvc outputs. The MSVC toolset is auto-selected by probe-msvc step (outputs: `selected_toolset`, `selected_full_version`, `install_path`). cl.exe full path: `$install_path\VC\Tools\MSVC\$selected_full_version\bin\HostX64\x64\cl.exe`.
- [Phase 2 note]: mamba install step uses `shell: pwsh` (not bash). Git Bash on Windows corrupts mamba channel arguments through conda's .bat activation chain. All conda/mamba steps should use pwsh.
- [Phase 2 note]: Step ordering — probe-msvc runs FIRST (before checkout/python/mamba) for instant fail on bad inputs. Mamba/CUDA install runs BEFORE any vcvars activation. The order matters for INCLUDE/LIB overflow prevention.
- [Phase 2 note]: Runner `windows-2022` has MSVC 14.29 + 14.44 (as of 2026-04-16). Auto-select picks 14.44 (_MSC_VER=1944) for CUDA 12.6 (cap=1949). CUDA 12.4-12.9 all accept any VS 2022 MSVC (_MSC_VER < 1950).

## Session Continuity

Last session: 2026-04-16T16:00:00Z
Stopped at: Completed 01-03-PLAN.md — Phase 1 complete (3/3 plans). SUMMARY written, STATE updated. Next: /gsd:verify-work for Phase 1, then Phase 2 planning.
Resume file: None
