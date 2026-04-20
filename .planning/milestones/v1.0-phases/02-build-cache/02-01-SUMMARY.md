---
phase: 02-build-cache
plan: 01
subsystem: infra
tags: [github-actions, ci, workflow, cuda, cmake, ninja, sccache, mamba, vcvarsall, scikit-build-core, wheel, windows, cache]

# Dependency graph
requires:
  - phase: 01-scaffold-toolchain-pinning
    provides: "Populated build job body (01-02/01-03, renamed from preflight): 15 named steps with step outputs (probe-msvc, assert-submodule, discover-vs) for build step consumption, forensics summary + remediation hint as terminal steps"
provides:
  - "Build body steps in .github/workflows/build-wheels-cuda-windows.yaml: VS BuildCustomizations copy, mamba cache restore, pip install build deps, sccache setup, vcvarsall+build (core build step), mamba cache save, sccache stats"
  - "Core build step: vcvarsall activation from probe-msvc outputs (no ilammy), CMAKE_ARGS with Ninja/CUDA/sccache/CMP0141, version override via __init__.py rewrite with PEP 440 local segment, python -m build --wheel"
  - "Split cache pattern: actions/cache/restore@v4 + actions/cache/save@v4 with if: always() for save-on-failure semantics (BLD-05/06)"
  - "sccache via hendrikmuhs/ccache-action@v1.2 with CUDA/MSVC-aware cache key"
  - "Step outputs: build-wheel.outputs.{wheel_version, wheel_path, wheel_name} for Phase 2 Plan 02 (assertions + upload)"
  - "Per-step timeout-minutes on every step in the job (BLD-09)"
affects: [02-02-assertions-upload, phase-3-smoke-test, phase-4-publish]

# Tech tracking
tech-stack:
  added:
    - "hendrikmuhs/ccache-action@v1.2 (sccache variant — handles binary install, PATH setup, GH Actions cache backend)"
    - "actions/cache/restore@v4 + actions/cache/save@v4 (split cache pattern for save-on-failure)"
    - "python -m build (PEP 517 frontend for scikit-build-core wheel production)"
    - "ninja (CMake generator — faster than MSBuild, sccache-compatible)"
  patterns:
    - "vcvarsall activation in pwsh: capture env before, run vcvarsall in cmd subprocess, apply diff to current session — avoids ilammy overflow and env persistence issues"
    - "Version override via __init__.py rewrite: regex-replace version before build, git checkout to restore after — works with scikit-build-core regex provider"
    - "Split cache restore/save: separate steps allow if: always() on save for fail-fix-retry loop affordability"
    - "CUDA tag computation: extract major.minor from cuda_version input, strip dot, prefix cu (12.6.3 -> cu126)"
    - "CMP0141 dual approach: CMAKE_POLICY_DEFAULT_CMP0141=NEW + CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded + /Z7 via FLAGS_INIT fallback (addresses sccache#2244)"

key-files:
  created: []
  modified:
    - ".github/workflows/build-wheels-cuda-windows.yaml (expanded from ~570 to ~866 lines: 7 new build steps inserted before forensics summary, timeout-minutes on every step, job timeout 20->90)"

key-decisions:
  - "vcvarsall activation INLINE in build step (not dedicated step) — env vars don't persist across GH Actions steps; avoids ilammy overflow"
  - "12-char llama.cpp SHA (re-derived via git rev-parse, not assert-submodule's 7-char output) for collision safety in large repo"
  - "CUDA tag computed dynamically from inputs.cuda_version (not hardcoded) — supports future CUDA version additions"
  - "CMAKE_BUILD_PARALLEL_LEVEL=2 (runner has 4 vCPUs but CUDA compilation is memory-heavy)"
  - "SKBUILD_WHEEL_PY_API='' to override pyproject.toml wheel.py-api='py3' — produces cpXX tag instead of py3-none"
  - "CMP0141 + /Z7 dual approach: policy variable in CMAKE_ARGS AND manual FLAGS_INIT fallback (sccache#2244 workaround)"
  - "Mamba cache key includes hashFiles of workflow file itself (not source files) — invalidates on workflow changes"
  - "sccache cache key includes cuda_version + msvc_toolset + hashFiles of llama.cpp source — partial restore falls back to cuda+msvc only"

patterns-established:
  - "Pattern 1: vcvarsall env capture — $envBefore hashtable, cmd /c vcvarsall && set, apply diff. Reusable for any step needing MSVC env."
  - "Pattern 2: __init__.py rewrite + git restore — modifies tracked file before build, restores after. Safe because scikit-build-core reads at configure time."
  - "Pattern 3: Save-on-failure cache — actions/cache/restore + actions/cache/save(if: always()) as separate steps. Apply to any expensive install."
  - "Pattern 4: CUDA tag from input — split('.')[0..1], join, prefix 'cu'. E.g. 12.6.3 -> cu126."

requirements-completed: [BLD-01, BLD-02, BLD-03, BLD-04, BLD-05, BLD-06, BLD-07, BLD-08, BLD-09, BLD-12]

# Metrics
duration: ~4min
completed: 2026-04-16
---

# Phase 2 Plan 1: Build Body and Cache Wiring Summary

**vcvarsall-activated CUDA wheel build via scikit-build-core/Ninja with sccache compiler launchers (C/CXX/CUDA), PEP 440 version embedding (0.3.20+cu126.ll<sha12>), split mamba cache with save-on-failure, and per-step timeouts on all 22 steps**

## Performance

- **Duration:** ~4 min
- **Started:** 2026-04-16T14:59:57Z
- **Completed:** 2026-04-16T15:04:20Z
- **Tasks:** 2
- **Files modified:** 1

## Accomplishments

- Added 7 new build steps to the build job (renamed from preflight), inserted before the forensics summary: VS BuildCustomizations copy, mamba cache restore, pip install build deps, sccache setup, core build step (vcvarsall + cmake + ninja + wheel), mamba cache save (if: always()), sccache stats (if: always()).
- Core build step activates MSVC via vcvarsall.bat in a clean pwsh subprocess using probe-msvc outputs (install_path + selected_full_version), constructs CMAKE_ARGS with Ninja generator, CUDA on, 5 architecture targets (80/86/89/90-real + 90-virtual), sccache launchers for C/CXX/CUDA, CMP0141/Z7 for sccache compatibility, CPU ISA disables (AVX2/FMA/F16C off), and builds the wheel with `python -m build --wheel`.
- Version string embeds llama.cpp submodule SHA in PEP 440 local segment format (`0.3.20+cu126.ll<sha12>`) via __init__.py rewrite before build, then git checkout to restore.
- Split mamba pkgs cache (actions/cache/restore@v4 + actions/cache/save@v4) with `if: always()` on save for fail-fix-retry loop affordability.
- sccache installed via hendrikmuhs/ccache-action@v1.2 with CUDA/MSVC-aware cache key (invalidates on source change, falls back on cuda+msvc match).
- Added `timeout-minutes` to every step in the job that was missing one (12 existing steps upgraded + 7 new steps). Total timeout-minutes declarations: 23.
- Job-level timeout increased from 20 to 90 minutes (cold CUDA build with 5 arch targets could take 60 min).

## Task Commits

Each task was committed atomically:

1. **Task 1: Add build dependencies, sccache setup, and mamba cache restore/save steps** - `2721479` (feat)
2. **Task 2: Add the core build step with vcvarsall activation, CMAKE_ARGS, version override, and wheel production** - `96153a4` (feat)

## Files Created/Modified

- `.github/workflows/build-wheels-cuda-windows.yaml` (MODIFIED, ~570 to ~866 lines) -- 7 new build steps inserted before forensics summary step. timeout-minutes added to all steps. Job timeout 20->90. Key additions: vcvarsall activation (Pattern 1), CMAKE_ARGS with 14 flags, PEP 440 version override (Pattern 2), split cache (Pattern 3), sccache with Ninja/CUDA/MSVC key.

## Decisions Made

### vcvarsall Inline (not dedicated step)
GitHub Actions env vars don't persist across steps. vcvarsall must run in the same pwsh process as the build command. This is documented in RESEARCH.md Pattern 1 and confirmed by Phase 1 STATE.md constraint.

### 12-char SHA (re-derived)
assert-submodule outputs a 7-char short SHA. For the 12-char requirement per CONTEXT.md, the build step re-derives via `git -C vendor/llama.cpp rev-parse HEAD` and takes `.Substring(0, 12)`.

### SKBUILD_WHEEL_PY_API empty string
pyproject.toml sets `wheel.py-api = "py3"` which would produce `py3-none-win_amd64` wheels. Setting `SKBUILD_WHEEL_PY_API=''` overrides this to get the default `cpXX-cpXX` tag. Fallback options documented inline: `-C wheel.py-api=""` config-setting, or explicit `cp311`.

### CMP0141 dual approach
CMAKE_POLICY_DEFAULT_CMP0141=NEW + CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded is the standard approach, but sccache#2244 reports it sometimes doesn't convert /Zi to /Z7. Added CMAKE_C_FLAGS_INIT="/Z7" and CMAKE_CXX_FLAGS_INIT="/Z7" as safety net.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## Authentication Gates

None -- no external service authentication required for this plan.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- **Ready for Plan 02 (assertions + upload):** Build step emits `wheel_version`, `wheel_path`, and `wheel_name` as step outputs. Plan 02 can add wheel tag assertion (BLD-10), wheel size assertion (BLD-11), and artifact upload (BLD-13) steps consuming these outputs.
- **Empirical validation needed on first dispatch:** SKBUILD_WHEEL_PY_API='' empty string behavior, CMP0141 effectiveness with scikit-build-core, sccache + nvcc + Ninja Windows interaction, mamba pkgs cache path. All have documented fallbacks.
- **No blockers.** All Phase 2 Plan 01 requirements (BLD-01 through BLD-09, BLD-12) are structurally complete pending dispatch verification.

---

## Self-Check: PASSED
