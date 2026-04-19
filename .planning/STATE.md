---
gsd_state_version: 1.0
milestone: shipped-v1.0
milestone_name: "(planning next milestone)"
status: between_milestones
stopped_at: "v1.0 Windows CUDA Wheels CI shipped 2026-04-19. Planning next milestone."
last_updated: "2026-04-19T17:10:00.000Z"
last_activity: "2026-04-19 — v1.0 milestone archived. Live Release v0.3.20-cu126-win with runtime-verified wheel. 4 phases / 8 plans / ~5 days. Archives: .planning/milestones/v1.0-{ROADMAP,REQUIREMENTS,MILESTONE-AUDIT}.md. Known gaps accepted as tech debt: Phase 3 VERIFICATION.md missing (evidence in dispatch runs), README.md:76 cosmetic stale, Nyquist partial on Phase 2/3."
progress:
  total_phases: 0
  completed_phases: 0
  total_plans: 0
  completed_plans: 0
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-19 after v1.0 milestone close)

**Core value:** Produce a Windows x64 CUDA wheel that actually works at runtime and is consumer-installable via manual download from GitHub Releases.
**Current focus:** Between milestones. v1.0 shipped; next milestone (v2 scope) not yet defined. Run `/gsd:new-milestone` to begin requirements/roadmap for v2.

## Current Position

No active phase. v1.0 Windows CUDA Wheels CI complete (2026-04-19). All 8 plans across 4 phases delivered; live Release v0.3.20-cu126-win shipped. See `.planning/MILESTONES.md` for v1.0 summary and `.planning/milestones/v1.0-MILESTONE-AUDIT.md` for the close-out audit.
Last activity: 2026-04-19 — v1.0 milestone archived via `/gsd:complete-milestone`.

Progress: [██████████] 100% (8 plans complete of 8 total: Phase 1: 3/3, Phase 2: 2/2, Phase 3: 1/1, Phase 4: 2/2)

## Performance Metrics

**Velocity:**
- Total plans completed: 5
- Average duration: ~6.5h (skewed by Phase 1 Plan 02's 6-dispatch empirical discovery cycle)
- Total execution time: ~32+ hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. Scaffold & Toolchain Pinning | 3/3 | ~28.1h | ~9.3h |
| 2. Build & Cache | 2/2 | ~multi-session | ~variable |
| 3. Smoke Test (Publish Gate) | 1/1 | ~multi-session | ~multi-session |
| 4. Publish & Consumer UX | 2/2 | ~8h 8m | ~4h |

**Recent Trend:**
- Last 5 plans: P1-02 (~24h multi-session), P1-03 (~4h), P2-01 (~4 min), P2-02 (multi-session with 6+ dispatch fix iterations)
- Trend: Phase 2 Plan 02 required multiple dispatch iterations for empirical discovery (VS BuildCustomizations, sccache OOM, timeouts, wheel tag, cache keys). This is expected for dispatch-verification checkpoints.

*Updated after each plan completion*
| Phase 01-scaffold-toolchain-pinning P01 | 2 min | 1 tasks | 1 files |
| Phase 01-scaffold-toolchain-pinning P02 | ~24h | 3 tasks | 1 files |
| Phase 01-scaffold-toolchain-pinning P03 | ~4h | 3 tasks | 1 files |
| Phase 02-build-cache P01 | ~4 min | 2 tasks | 1 files |
| Phase 02-build-cache P02 | ~multi-session | 2 tasks | 1 files |
| Phase 04-publish-consumer-ux P01 | ~8 min | 3 tasks | 4 files |
| Phase 04-publish-consumer-ux P02 | ~8h (dispatch 2h1m + remediation) | 2 tasks | 2 files |

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

- [Phase 02-build-cache 2026-04-16]: Job renamed `preflight` -> `build` in workflow YAML. The job now contains both toolchain preflight assertions AND build steps, so "preflight" was misleading. All future references should use job ID `build` (not `preflight`).
- [Phase 02-build-cache 2026-04-17]: `-C wheel.py-api=""` config-setting on `python -m build` is the CORRECT way to override pyproject.toml's `wheel.py-api = "py3"`. The `SKBUILD_WHEEL_PY_API=''` env var approach does NOT work because scikit-build-core ignores empty string environment variables.
- [Phase 02-build-cache 2026-04-17]: CMAKE_BUILD_PARALLEL_LEVEL=1 (not 2). Parallel nvcc with 5 CUDA arch targets kills the sccache daemon (OS error 10054 / connection reset) on the 4-vCPU runner. Sequential is slower (~120 min cold) but reliable.
- [Phase 02-build-cache 2026-04-17]: sccache `append-timestamp: false` is REQUIRED for cache reuse across runs. Without it, hendrikmuhs/ccache-action appends a timestamp making every cache key unique.
- [Phase 02-build-cache 2026-04-17]: Build step timeout must be 180 min (cold CUDA build with 5 arch targets + parallel=1). Job timeout 210 min. Original estimates (60/90) were far too low.
- [Phase 02-build-cache 2026-04-17]: Phase 2 COMPLETE. End-to-end verified: `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` (358 MB). All 13 BLD-* requirements green. Artifact name `cuda-wheel` is the contract for Phase 3 download-artifact.

- [Phase 02-build-cache 2026-04-16]: VS BuildCustomizations (CUDA .props/.targets) come from Jimver CUDA installer download + 7-Zip extraction, NOT from mamba's cuda-toolkit package (mamba does not ship visual_studio_integration files). Matches the old build-wheels-cuda.yaml approach. Not cached per CONTEXT.md — download+extract ~12s per run.
- [Phase 02-build-cache 2026-04-16]: vcvarsall activation INLINE in build step (not dedicated step) -- env vars don't persist across GH Actions steps. Uses probe-msvc outputs (install_path + selected_full_version) to locate vcvarsall.bat.
- [Phase 02-build-cache 2026-04-16]: 12-char llama.cpp SHA re-derived via `git -C vendor/llama.cpp rev-parse HEAD` (assert-submodule only outputs 7 chars). Collision-safe for large repo.
- [Phase 02-build-cache 2026-04-16]: SKBUILD_WHEEL_PY_API='' overrides pyproject.toml wheel.py-api='py3' to produce cpXX tag. Fallback: -C wheel.py-api="" or explicit cp311. Needs empirical validation.
- [Phase 02-build-cache 2026-04-16]: CMP0141 dual approach -- CMAKE_POLICY_DEFAULT_CMP0141=NEW + Embedded + /Z7 FLAGS_INIT fallback (sccache#2244 workaround). Empirical validation on first dispatch.
- [Phase 02-build-cache 2026-04-16]: CMAKE_BUILD_PARALLEL_LEVEL=2 (runner has 4 vCPUs but CUDA compilation is memory-heavy).
- [Phase 04-publish-consumer-ux]: 2026-04-19 scope reduction: dropped gh-pages PEP 503 pip index (PUB-03..PUB-10 + DOC-03) in favour of Release-only publishing with manual download; applied via REQUIREMENTS.md Out of Scope move + PROJECT.md Key Decisions row
- [Phase 04-publish-consumer-ux]: 12-char llama.cpp SHA for Release body forensics parsed from wheel filename (option b of SHA-width fix), NOT via new assert-submodule output — keeps Phase 1 step untouched
- [Phase 04-publish-consumer-ux]: Lint-workflow publish-gate uses anchored ^    needs[:]... pattern so prose/comments mentioning the phrase don't trigger the exactly-1-match invariant; error strings paraphrased ('colon'/'dash') to stay regex-invisible
- [Phase 04-publish-consumer-ux]: 2026-04-19 Plan 04-02 dispatch-verification complete. Dispatched against windows-build branch (both branches now in sync on origin); release v0.3.20-cu126-win live at target commit 973aeff with wheel SHA-256 47649ad0...adbeed8. 12-char llama.cpp SHA 227ed28e128e rendered in body narrative + forensics table + wheel filename (DOC-05 satisfied).
- [Phase 04-publish-consumer-ux]: 2026-04-19 Release body content gap: CUDA-runtime-bundling disclosure added via two-path remediation — live release patched via gh release edit (d52a043) AND workflow template patched so future dispatches render the disclosure natively (d52a043) + forensics re-captured (1d1876b). Prevents consumers from assuming they need a local CUDA Toolkit install.

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

Last session: 2026-04-19T10:19:19.008Z
Stopped at: Completed 04-02-PLAN.md (green end-to-end dispatch + live release v0.3.20-cu126-win); Phase 4 ready for /gsd:verify-work
Resume file: None
