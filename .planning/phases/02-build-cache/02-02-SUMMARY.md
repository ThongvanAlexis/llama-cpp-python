---
phase: 02-build-cache
plan: 02
subsystem: infra
tags: [github-actions, ci, workflow, cuda, wheel-tag, wheel-size, upload-artifact, assertions, windows, dispatch-verification]

# Dependency graph
requires:
  - phase: 02-build-cache
    plan: 01
    provides: "Build step with wheel_version/wheel_path/wheel_name outputs, vcvarsall activation, CMAKE_ARGS, sccache, mamba cache, per-step timeouts"
provides:
  - "Post-build wheel tag regex assertion (BLD-10): fails job if wheel is not cp311-cp311-win_amd64"
  - "Post-build wheel size assertion (BLD-11): fails job if wheel >= 400 MB"
  - "Artifact upload via actions/upload-artifact@v4 (BLD-13): cuda-wheel artifact consumable by Phase 3 smoke-test download-artifact"
  - "Forensics summary extended with wheel version and filename rows"
  - "End-to-end verified: dispatched build produces correct wheel (358 MB, cp311 tag, +cu126.ll<sha12> version)"
affects: [phase-3-smoke-test, phase-4-publish]

# Tech tracking
tech-stack:
  added:
    - "actions/upload-artifact@v4 (wheel artifact upload with compression-level: 0, retention-days: 7)"
  patterns:
    - "Wheel tag regex guard: derive expected cpXX from inputs.python_version, assert exactly 1 wheel in dist/, assert filename matches pattern, emit remediation hints for common mismatches (py3-none, abi3, none-any)"
    - "Wheel size budget assertion: fail-fast if wheel exceeds 400 MB, with remediation suggesting CMAKE_CUDA_ARCHITECTURES reduction or debug symbol check"
    - "Artifact name contract: 'cuda-wheel' is the stable name consumed by Phase 3 download-artifact"

key-files:
  created: []
  modified:
    - ".github/workflows/build-wheels-cuda-windows.yaml (3 new assertion/upload steps after build, forensics summary extended with wheel version/filename rows, job renamed preflight->build, timeout bumped 90->210min)"

key-decisions:
  - "Job renamed preflight -> build: job now contains both toolchain preflight AND build steps, so 'preflight' was misleading"
  - "VS BuildCustomizations from Jimver CUDA installer download + 7-Zip extraction, NOT mamba (mamba does not ship visual_studio_integration files)"
  - "-C wheel.py-api='' config-setting override (not SKBUILD_WHEEL_PY_API env var): env var empty string is ignored by scikit-build-core; -C config-setting works"
  - "CMAKE_BUILD_PARALLEL_LEVEL=1 (not 2): parallel nvcc kills sccache daemon with OS error 10054 on the 4-vCPU runner"
  - "sccache cache key: append-timestamp: false required for stable cache keys across runs"
  - "Build step timeout: 180 min (not 60): cold CUDA build with 5 arch targets + sequential nvcc takes ~120 min"
  - "Job timeout: 210 min (not 90): accommodates 180 min build + preflight + post-build steps"
  - "nvcc -diag-suppress=177,221,550: suppress unused variable/parameter warnings that flood the build log"

patterns-established:
  - "Pattern 1: Wheel tag regex guard — derive cpXX from dispatch input, assert filename matches, emit targeted remediation for py3-none/abi3/none-any mismatches"
  - "Pattern 2: Wheel size budget — hard fail at 400 MB with remediation suggesting arch reduction or debug symbol check"
  - "Pattern 3: Artifact name contract — 'cuda-wheel' is the stable interface between build and smoke-test jobs"

requirements-completed: [BLD-10, BLD-11, BLD-13]

# Metrics
duration: ~multi-session (Task 1: ~5 min code, Task 2: ~several hours across dispatch iterations)
completed: 2026-04-17
---

# Phase 2 Plan 2: Post-Build Assertions, Artifact Upload, and Dispatch Verification Summary

**Wheel tag regex guard (cp311-cp311-win_amd64), 400 MB size assertion, upload-artifact@v4 with cuda-wheel contract, and end-to-end dispatch verification proving the complete build pipeline produces a correct 358 MB wheel**

## Performance

- **Duration:** Multi-session (Task 1 code: ~5 min; Task 2 dispatch verification: several hours across 6+ fix iterations)
- **Started:** 2026-04-16 (continued from Plan 01 completion)
- **Completed:** 2026-04-17
- **Tasks:** 2
- **Files modified:** 1

## Accomplishments

- Added 3 post-build steps to the workflow: wheel tag assertion (BLD-10), wheel size assertion (BLD-11), and artifact upload via upload-artifact@v4 (BLD-13). All three have explicit timeout-minutes.
- Wheel tag assertion derives expected Python tag from dispatch input (`'3.11' -> '311'`), asserts exactly 1 wheel in `dist/`, validates filename against `cp${pyVer}-cp${pyVer}-win_amd64\.whl$` regex, and provides targeted remediation hints for common mismatches (py3-none, abi3, none-any).
- Wheel size assertion enforces 400 MB budget with remediation suggesting CMAKE_CUDA_ARCHITECTURES reduction or debug symbol check.
- Artifact upload uses `compression-level: 0` (wheel is already compressed), `retention-days: 7`, `if-no-files-found: error`, and the artifact name `cuda-wheel` which is the contract for Phase 3 smoke-test job's download-artifact step.
- Extended forensics summary with wheel version and wheel filename rows from build-wheel step outputs.
- End-to-end dispatch verified: complete pipeline produces `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` (358 MB) with all assertion steps green, cuda-wheel artifact uploaded.

## Task Commits

Each task was committed atomically:

1. **Task 1: Add wheel tag assertion, wheel size assertion, and artifact upload steps** - `904c5cd` (feat)
2. **Task 2: Dispatch and verify complete build pipeline end-to-end** - checkpoint:human-verify (approved after fix iterations)

**Fix commits during dispatch verification (Rule 1/3 auto-fixes):**
- `454fa95` - fix: rename job preflight->build, fix VS BuildCustomizations source (mamba doesn't ship them)
- `e5ada5e` - fix: suppress nvcc warnings (-diag-suppress=177,221,550), drop parallel level to 1 (sccache OOM)
- `7d05a1e` - fix: bump build step timeout 60->180min, job timeout 90->210min
- `a91473c` - fix: use `-C wheel.py-api=""` config-setting for correct cpXX wheel tag (SKBUILD_WHEEL_PY_API env var empty string ignored by scikit-build-core)
- `77f55fd` - fix: sccache cache key needs `append-timestamp: false` for stable keys

## Files Created/Modified

- `.github/workflows/build-wheels-cuda-windows.yaml` (MODIFIED) -- 3 new post-build steps (tag assertion, size assertion, artifact upload), forensics summary extended with wheel version/filename rows. Fix iterations: job rename preflight->build, VS BuildCustomizations source corrected, nvcc warning suppression, parallel level 1, timeout bumps, wheel.py-api config-setting, sccache append-timestamp fix.

## Decisions Made

### Job renamed preflight -> build
The single job now contains both Phase 1 preflight assertions AND Phase 2 build/assertion steps. Naming it "preflight" was misleading since it does far more than preflight checks. All downstream references updated.

### VS BuildCustomizations from Jimver installer (not mamba)
mamba's `cuda-toolkit` package does NOT include `visual_studio_integration/MSBuildExtensions` (the .props/.targets files needed for MSBuild/Ninja CUDA compilation). These only come from the NVIDIA CUDA installer .exe. Adopted the same pattern as upstream's build-wheels-cuda.yaml: download from Jimver's link table, extract with 7-Zip, copy to VS BuildCustomizations directory.

### -C wheel.py-api="" (not SKBUILD_WHEEL_PY_API env var)
pyproject.toml sets `wheel.py-api = "py3"` which produces `py3-none-win_amd64` wheels. The original plan to use `SKBUILD_WHEEL_PY_API=''` env var failed because scikit-build-core ignores empty string environment variables. The `-C wheel.py-api=""` config-setting passed to `python -m build` works correctly, producing `cp311-cp311-win_amd64` tags.

### CMAKE_BUILD_PARALLEL_LEVEL=1 (not 2)
Runner has 4 vCPUs but parallel nvcc with 5 architecture targets kills the sccache daemon with OS error 10054 (connection reset by peer). Sequential compilation (parallel=1) is slower but reliable. Build takes ~120 min cold.

### sccache append-timestamp: false
Without this, hendrikmuhs/ccache-action appends a timestamp to the cache key on every run, preventing cache reuse across runs. Setting `append-timestamp: false` produces stable keys.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Job rename preflight -> build**
- **Found during:** Task 2 (dispatch verification)
- **Issue:** Job ID was still `preflight` but now contains build steps; also confusing for downstream `needs:` references
- **Fix:** Renamed job ID from `preflight` to `build` throughout the workflow
- **Files modified:** .github/workflows/build-wheels-cuda-windows.yaml
- **Committed in:** 454fa95

**2. [Rule 3 - Blocking] VS BuildCustomizations source corrected**
- **Found during:** Task 2 (dispatch verification, first dispatch failed)
- **Issue:** mamba's cuda-toolkit does not ship visual_studio_integration/MSBuildExtensions files; build step could not find CUDA .props/.targets
- **Fix:** Added step to download CUDA installer from Jimver's link table, extract MSBuildExtensions with 7-Zip, copy to VS BuildCustomizations directory
- **Files modified:** .github/workflows/build-wheels-cuda-windows.yaml
- **Committed in:** 454fa95

**3. [Rule 1 - Bug] nvcc warning suppression and parallel level fix**
- **Found during:** Task 2 (dispatch, sccache daemon crash)
- **Issue:** Parallel nvcc compilation killed sccache daemon (OS error 10054); nvcc warnings flooded build log
- **Fix:** Added `-diag-suppress=177,221,550` to CMAKE_CUDA_FLAGS; dropped CMAKE_BUILD_PARALLEL_LEVEL from 2 to 1
- **Files modified:** .github/workflows/build-wheels-cuda-windows.yaml
- **Committed in:** e5ada5e

**4. [Rule 1 - Bug] Build step timeout insufficient**
- **Found during:** Task 2 (dispatch, build timed out at 60 min)
- **Issue:** Cold CUDA build with 5 arch targets and parallel=1 takes ~120 min; 60 min timeout was too low
- **Fix:** Build step timeout 60->180min, job timeout 90->210min
- **Files modified:** .github/workflows/build-wheels-cuda-windows.yaml
- **Committed in:** 7d05a1e

**5. [Rule 1 - Bug] Wheel tag incorrect (py3-none instead of cp311)**
- **Found during:** Task 2 (dispatch, wheel tag assertion failed)
- **Issue:** SKBUILD_WHEEL_PY_API='' env var empty string is ignored by scikit-build-core; pyproject.toml's wheel.py-api='py3' still took effect producing py3-none tag
- **Fix:** Switched from env var to `-C wheel.py-api=""` config-setting on `python -m build` command line
- **Files modified:** .github/workflows/build-wheels-cuda-windows.yaml
- **Committed in:** a91473c

**6. [Rule 1 - Bug] sccache cache key instability**
- **Found during:** Task 2 (dispatch, sccache cache never hit)
- **Issue:** hendrikmuhs/ccache-action defaults to `append-timestamp: true`, making cache keys unique per run (no reuse)
- **Fix:** Added `append-timestamp: false` to sccache step configuration
- **Files modified:** .github/workflows/build-wheels-cuda-windows.yaml
- **Committed in:** 77f55fd

---

**Total deviations:** 6 auto-fixed (4 Rule 1 bugs, 2 Rule 3 blocking issues)
**Impact on plan:** All auto-fixes were discovered during dispatch verification and were necessary for the build pipeline to produce a correct wheel. The dispatch verification checkpoint (Task 2) is specifically designed for this empirical discovery cycle. No scope creep.

## Issues Encountered

- Multiple dispatch iterations required to discover and fix empirical issues that could not be predicted from documentation alone: mamba's lack of VS integration files, sccache daemon OOM on parallel nvcc, scikit-build-core ignoring empty env var overrides, build timeout underestimation. Each was fixed and committed individually. This is expected behavior for the verification checkpoint -- the plan anticipated that "empirical validation needed on first dispatch" (from Plan 01 Summary).

## Authentication Gates

None -- no external service authentication required.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- **Phase 2 COMPLETE.** All 13 BLD-* requirements (BLD-01 through BLD-13) are verified by a green dispatch run.
- **Ready for Phase 3 (Smoke Test):** The `cuda-wheel` artifact is uploaded via upload-artifact@v4 and consumable by Phase 3's download-artifact@v4 step. Wheel filename, version, and tag are verified.
- **sccache cache primed:** The first cold run saved the sccache cache. Subsequent rebuilds (after YAML-only changes) should see >30% hit rate and faster builds.
- **mamba pkgs cache primed:** Saved with `if: always()` semantics.
- **No blockers.** Phase 3 can begin immediately.

---

## Self-Check: PASSED

All files exist, all 6 commits verified, workflow contains upload-artifact@v4, cuda-wheel artifact name, wheel tag regex, and 400 MB size limit.

---
*Phase: 02-build-cache*
*Completed: 2026-04-17*
