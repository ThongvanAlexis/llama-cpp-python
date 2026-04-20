---
phase: 03-smoke-test-publish-gate
plan: 01
subsystem: infra
tags: [github-actions, ci, workflow, cuda, smoke-test, dll-load, wheel-verification, windows, dispatch-verification, scikit-build-core, ggml-native, nvcuda-stub]

# Dependency graph
requires:
  - phase: 02-build-cache
    plan: 02
    provides: "cuda-wheel artifact (upload-artifact@v4), wheel tag regex, wheel size assertion, +cu126.ll<sha12> version embed"
provides:
  - "smoke-test job on fresh windows-2022 runner with needs: [build] dependency"
  - "Sparse checkout (cone-mode: false) excluding llama_cpp/ source so import resolves to site-packages"
  - "Clean venv + pip install from cuda-wheel artifact"
  - "nvcuda.dll stub for driverless runners (pefile-discovered exports)"
  - "cuobjdump CUDA architecture verification gate (sm_80/86/89/90)"
  - "DLL-load import gate (from llama_cpp import Llama, llama_cpp) — catches upstream #1543"
  - "Green forensics summary + red diagnostics step"
  - "Mamba pkgs cache with separate mamba-smoke-* key prefix"
  - "GGML_NATIVE=OFF for portable /arch:AVX wheels regardless of build runner CPU"
  - "CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL forcing Release CRT independent of scikit-build-core's Debug-default"
  - "CMAKE_{C,CXX,CUDA}_FLAGS_DEBUG override stripping /Od /RTC1 from Debug config"
  - "End-to-end verified green dispatch: build + smoke-test + lint-workflow all pass"
affects: [phase-4-publish]

# Tech tracking
tech-stack:
  added:
    - "actions/download-artifact@v4 (smoke-test consumes cuda-wheel)"
    - "conda-incubator/setup-miniconda@v3.1.0 in smoke-test (separate env: smoke-test, not llamacpp)"
    - "actions/cache@v4 for mamba-smoke-* key prefix"
    - "pefile (python) — runtime DLL dependency audit on failure + nvcuda export discovery for stub"
  patterns:
    - "nvcuda.dll stub: pefile-scan wheel DLLs + CUDA toolkit bin for IMAGE_DIRECTORY_ENTRY_IMPORT referencing nvcuda.dll; generate C stub exporting every discovered function returning CUDA_ERROR_NO_DEVICE (100); cl /O2 /LD build; copy to CONDA_PREFIX/Library/bin"
    - "DLL-load gate pattern: ctypes-backed Python import (`from llama_cpp import Llama, llama_cpp`) triggers full DLL load chain (llama.dll -> ggml.dll -> ggml-cuda.dll -> cudart/cublas/nvcuda). Non-zero exit => red smoke-test. No need for GPU on runner."
    - "pyproject.toml CI-time patching: rewrite minimum-version/cmake.minimum-version/cmake.build-type; restore via `git checkout --` after build. Same pattern as the __init__.py version embed from Phase 2."
    - "CMAKE_MSVC_RUNTIME_LIBRARY escape hatch: scikit-build-core strips -DCMAKE_BUILD_TYPE from CMAKE_ARGS and ignores every cmake.build-type override channel, but CMAKE_MSVC_RUNTIME_LIBRARY is passed through, directly selecting /MD vs /MDd independent of BUILD_TYPE."
    - "CMAKE_*_FLAGS_DEBUG override: override the Debug-config flag profile (/Od /RTC1 -> /O2 /Ob2 /DNDEBUG) instead of fighting scikit-build-core's Debug default."
    - "GGML_NATIVE=OFF portability: disables ggml-cpu's FindSIMD.cmake auto-detection on MSVC, producing /arch:AVX baseline binaries that run on any x86-64 CPU regardless of which Azure VM the build runner lands on."

key-files:
  created: []
  modified:
    - ".github/workflows/build-wheels-cuda-windows.yaml (smoke-test job added between build and lint-workflow; CMAKE_ARGS extended with GGML_NATIVE=OFF, CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL, CMAKE_POLICY_DEFAULT_CMP0091=NEW, CMAKE_*_FLAGS_DEBUG overrides; CI-time pyproject.toml patch step; pyproject.toml restore via git checkout after build)"
  deleted:
    - "llm_for_ci_test/tiny-llama.gguf (fixture no longer referenced after smoke-test simplification)"
    - "llm_for_ci_test/Readme.md (accompanying notes for deleted fixture, untracked)"

key-decisions:
  - "Smoke-test simplified from CPU-only inference to DLL-load import gate: wheels are shipped for GPU users (n_gpu_layers=-1) who never exercise ggml-cpu hot paths; forcing CPU inference on the runner surfaced failures (tiny-llama.gguf BPE edge cases, host-native /arch:AVX512 crashes) unrelated to real-world usage. DLL-load gate still catches upstream #1543 (wheel compiles but crashes on import) without false-fail surface."
  - "tiny-llama.gguf fixture removed: after simplification to DLL-load gate, the committed 27 KB pathological GGUF (n_vocab=32, n_layer=1) is unreferenced. Deleted along with its Readme."
  - "Release build type enforcement via CMAKE_MSVC_RUNTIME_LIBRARY + CMAKE_*_FLAGS_DEBUG override: scikit-build-core refuses every channel for CMAKE_BUILD_TYPE=Release (pyproject.toml TOML key, -C config-setting, SKBUILD_CMAKE_BUILD_TYPE env var, CMAKE_ARGS). Gave up on flipping build-type and instead force-selected Release CRT and Release-like Debug-config flags via direct CMake variable assignments that ARE honored."
  - "GGML_NATIVE=OFF for portability: llama.cpp's ggml-cpu/CMakeLists.txt runs FindSIMD.cmake on MSVC with GGML_NATIVE=ON (default), baking /arch:AVX512 into ggml-cpu.dll whenever the build runner has AVX-512. GitHub-hosted Windows runners don't guarantee stable CPU SKUs, so build-time CPU capabilities leaked into wheels non-deterministically. OFF forces an /arch:AVX baseline that works on any x86-64 CPU from Sandy Bridge/Jaguar (2011+)."
  - "nvcuda.dll stub via pefile auto-discovery: GitHub runners have no NVIDIA GPU driver, so nvcuda.dll is absent, breaking DllMain of ggml-cuda.dll. Stub is built at CI time by pefile-scanning both the CUDA toolkit DLLs AND the wheel's DLLs (llama_cpp/lib/*.dll) for IMAGE_DIRECTORY_ENTRY_IMPORT entries referencing nvcuda.dll, generating a C stub that exports every discovered function, and compiling with cl.exe /O2 /LD. Ensures the stub remains correct across llama.cpp versions."
  - "pyproject.toml patched at CI time (not upstream): minimum-version 0.5.1 -> 0.9.2 (exits scikit-build-core compat mode), cmake.minimum-version -> cmake.version (deprecated key rename), and cmake.build-type = 'Release' injected. All restored via `git checkout -- pyproject.toml` post-build so the tracked file is unchanged."
  - "CPU-only inference test is NOT in the final gate. The plan's ST-06 (2-token generation on tiny-llama.gguf) was superseded by the import-only gate after discovering that (a) the committed fixture was pathologically degenerate and (b) the GPU users this fork is built for never exercise the CPU code path."

patterns-established:
  - "Pattern 1: DLL-load import gate — the minimum-viable runtime verification for CUDA wheels on a GPU-less CI runner. ctypes-backed Python import triggers full DLL chain; non-zero exit => red. No GGUF fixture, no inference, no CPU-path exercise."
  - "Pattern 2: nvcuda.dll stub via pefile auto-discovery — forward-compatible stub generation that scans both CUDA toolkit and wheel DLLs for imports from nvcuda.dll and synthesizes a returning-CUDA_ERROR_NO_DEVICE stub."
  - "Pattern 3: CMAKE_MSVC_RUNTIME_LIBRARY escape hatch — when scikit-build-core refuses every channel to flip CMAKE_BUILD_TYPE, force Release CRT directly via CMAKE_MSVC_RUNTIME_LIBRARY + CMP0091=NEW and strip Debug-only codegen via CMAKE_*_FLAGS_DEBUG overrides."
  - "Pattern 4: GGML_NATIVE=OFF on CI — always required for portable wheels on CI with non-deterministic runner CPU SKUs."
  - "Pattern 5: CI-time pyproject.toml patching — same restore-via-git-checkout pattern as the __init__.py version embed from Phase 2."

requirements-completed: [ST-02, ST-03, ST-04, ST-05, ST-07]
requirements-superseded:
  - "ST-01 (commit GGUF fixture): no fixture needed after simplification to DLL-load gate; fixture deleted"
  - "ST-06 (run inference on fixture): replaced with DLL-load import gate that triggers the same native DLL chain but does not exercise CPU hot paths"
requirements-deferred:
  - "ST-08 (publish job needs: [build, smoke-test] gate): Phase 4 will add the publish job; smoke-test job's existence and contract are in place for Phase 4 to depend on"

# Metrics
duration: multi-session (code: ~15 min over a few passes; dispatch-verification: ~16 iterations over 2 days)
completed: 2026-04-18
---

# Phase 3 Plan 1: Smoke-Test Publish Gate Summary

**DLL-load import gate on fresh windows-2022 runner, CUDA arch verification via cuobjdump, nvcuda.dll driver stub, Release CRT enforcement, portable /arch:AVX wheels, end-to-end dispatch-verified green.**

## Performance

- **Duration:** Multi-session (code edits ~15 min; dispatch verification ~16 iterations across ~2 days)
- **Started:** 2026-04-17
- **Completed:** 2026-04-18
- **Tasks:** 2 (Task 1: smoke-test job; Task 2: dispatch-verify human-checkpoint)
- **Files modified:** 1 (`.github/workflows/build-wheels-cuda-windows.yaml`)
- **Files deleted:** 2 (`llm_for_ci_test/tiny-llama.gguf` tracked, `llm_for_ci_test/Readme.md` untracked)

## Accomplishments

- Added `smoke-test` job on fresh `windows-2022` runner with `needs: [build]`, timeout-minutes, and all standard conventions (shell: pwsh, Set-StrictMode, $ErrorActionPreference=Stop, remediation hints).
- Sparse-checkout pattern `!llama_cpp/` with `cone-mode: false` excludes repo source so `from llama_cpp import Llama` resolves to venv site-packages (proven by import-path assertion).
- `actions/download-artifact@v4` pulls `cuda-wheel` artifact produced by Phase 2 build job.
- Clean venv (`python -m venv .smoke-venv`) + `pip install <wheel>` into the venv's python.
- nvcuda.dll stub generation: pefile scans both CUDA toolkit DLLs and wheel DLLs (`llama_cpp/lib/*.dll`) for `nvcuda.dll` imports, generates a C stub exporting every discovered function, compiles with `cl.exe /O2 /LD`, copies into `CONDA_PREFIX/Library/bin`. Required because GitHub runners have no NVIDIA GPU driver.
- Asserts: `llama_cpp/__init__.py` absent from checkout, installed in site-packages; all expected wheel DLLs present (`llama.dll, ggml.dll, ggml-base.dll, ggml-cpu.dll, ggml-cuda.dll`); wheel version matches `+cu126.ll[a-f0-9]+` from filename.
- cuobjdump CUDA arch verification: `cuobjdump --list-elf ggml-cuda.dll` asserted to contain `sm_80, sm_86, sm_89, sm_90`.
- DLL-load import gate: `from llama_cpp import Llama, llama_cpp` in a subprocess; non-zero exit => red smoke-test. Catches upstream #1543 (wheel compiles but crashes on import) without exercising CPU hot paths.
- Green forensics summary table and red diagnostics step with pefile-backed DLL dependency audit on failure (walks imports of ggml-cuda.dll / ggml.dll / llama.dll and reports any missing exports).
- Mamba pkgs cache with separate `mamba-smoke-${{ inputs.cuda_version }}-${{ runner.os }}-<yaml-hash>` key prefix (distinct from build job's cache).
- WER (Windows Error Reporting) suppression via registry + `SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX)` P/Invoke so access violations return non-zero exit instead of hanging on a modal dialog.
- End-to-end dispatch-verified: run `24611962164` produced wheel `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` (340 MB), all three jobs green (build + smoke-test + lint-workflow), import test `import_ok=True`, cuobjdump archs `sm_80, sm_86, sm_89, sm_90`.

## Task Commits

Task 1 was committed atomically; Task 2 (human-verify dispatch) took many fix iterations as the dispatch loop surfaced empirical issues.

1. **Task 1: Add smoke-test job and GGUF fixture** — `649d66b` (feat)
2. **Task 2: Dispatch-verify** — extensive fix iterations, captured below.

**Fix commits during dispatch verification** (chronological, grouped by theme):

### CUDA DLL chain / nvcuda stub (5 commits)
- `44ed106` — export CUDA DLL path for venv steps (llama-cpp-python's `_ctypes_extensions.py` checks `CUDA_PATH`)
- `5bd3394` — set `CUDA_PATH` and add DLL-load forensics to smoke-test
- `bcfb503` — add `site-packages/bin` to DLL search path, fix forensics, add cache alert
- `202a70b` — create nvcuda.dll stub (runners have no GPU driver)
- `7edede5` — auto-discover nvcuda.dll exports (initial single-file discovery)
- `17d3128` — use pefile to discover nvcuda imports, fallback to `cuda.lib`
- `1b10a13` — add pefile deep dependency audit to find exact missing procedure
- `0ccf11f` — replace import assertion with file checks, move forensics to inference step
- `2b96812` — fix site-packages path in cuobjdump and red diagnostics
- `79d2c0e` — scan WHEEL DLLs for nvcuda imports, not just CUDA toolkit (the wheel's own ggml-cuda.dll also imports from nvcuda.dll)

### CRT / Build-type fight (7 commits)
- `cfbe73b` — increase sccache to 5G, attempt `CMAKE_BUILD_TYPE=Release` via CMAKE_ARGS (stripped)
- `8677982` — set build type via `-C cmake.build-type=Release` (silently ignored)
- `98590db` — force Release via `SKBUILD_CMAKE_BUILD_TYPE` env var (silently ignored)
- `9eddb01` — bump scikit-build-core `minimum-version` via pyproject.toml patch (surfaces deprecated cmake.minimum-version)
- `0830df7` — also rename deprecated `cmake.minimum-version` → `cmake.version` (still Debug)
- `08500ae` — force Release CRT via `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` + `CMP0091=NEW` (CRT landed, but still Debug codegen with `/Od /RTC1`)
- `b34a230` — inject `cmake.build-type = "Release"` into pyproject.toml directly (scikit-build-core still ignores it)
- `c3e48b8` — strip `/RTC1 /Od` by overriding `CMAKE_{C,CXX,CUDA}_FLAGS_DEBUG`

### Fixture / inference model (1 commit)
- `6a79267` — swap tiny-llama.gguf fixture to Qwen3.5-0.8B-Q8_0 via HF cache (cleared the 0xe06d7363 C++ throw)

### Portability / final simplification (2 commits)
- `5f39e6a` — disable GGML_NATIVE for portable wheels (fixes STATUS_ILLEGAL_INSTRUCTION from /arch:AVX512 on non-AVX-512 runners)
- `50fa0d9` — simplify smoke-test to DLL-load import gate; remove HF cache, GGUF download, inference test; delete tiny-llama.gguf fixture

## Files Created/Modified

- `.github/workflows/build-wheels-cuda-windows.yaml` (MODIFIED) — smoke-test job (17 steps) added between build and lint-workflow. CMAKE_ARGS extended with `GGML_NATIVE=OFF`, `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL`, `CMAKE_POLICY_DEFAULT_CMP0091=NEW`, `CMAKE_POLICY_DEFAULT_CMP0141=NEW`, `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded`, `CMAKE_{C,CXX}_FLAGS_INIT="/Z7"`, `CMAKE_{C,CXX,CUDA}_FLAGS_DEBUG="/O2 /Ob2 /DNDEBUG"`, `CMAKE_CUDA_FLAGS=-diag-suppress=177,221,550`. CI-time pyproject.toml patch step + git-checkout restore. Python build invoked with `python -m build --wheel -C wheel.py-api="" -C cmake.build-type=Release`.
- `llm_for_ci_test/tiny-llama.gguf` (DELETED) — committed GGUF fixture no longer referenced after simplification.

## Decisions Made

### Smoke-test simplified from CPU-inference to DLL-load import gate
The plan's ST-06 called for 2-token inference on a GGUF fixture with `n_gpu_layers=0`. In practice, this exercised `ggml-cpu.dll` hot paths that end users running with `n_gpu_layers=-1` (GPU offload) never execute. Two successive false-failures (BPE edge case on the pathologically degenerate tiny-llama.gguf, then STATUS_ILLEGAL_INSTRUCTION from `/arch:AVX512` baked into ggml-cpu.dll) were unrelated to real-world wheel quality. The DLL-load import gate (`from llama_cpp import Llama, llama_cpp`) still catches upstream #1543 (wheel compiles but crashes on import) — the original failure mode this phase exists to prevent — without the CPU-path false-fail surface. User approved this simplification in session.

### Release build type enforced via CMAKE_MSVC_RUNTIME_LIBRARY + CMAKE_*_FLAGS_DEBUG override (not via flipping BUILD_TYPE)
scikit-build-core silently ignores every documented channel for `cmake.build-type=Release`: pyproject.toml TOML key, `python -m build -C cmake.build-type=Release`, `SKBUILD_CMAKE_BUILD_TYPE` env var, and `-DCMAKE_BUILD_TYPE=Release` in CMAKE_ARGS (explicitly stripped with "Unsupported CMAKE_ARGS ignored" warning). Gave up on flipping BUILD_TYPE and instead: (a) `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` + `CMP0091=NEW` forces /MD (Release CRT) at link time, and (b) `CMAKE_{C,CXX,CUDA}_FLAGS_DEBUG` overrides replace the Debug profile (`/Od /RTC1`) with Release-like codegen (`/O2 /Ob2 /DNDEBUG`). Confirmed in build log: actual `cl.exe` invocation has `/O2 /Ob2 /DNDEBUG -MD` and no `/RTC1`.

### GGML_NATIVE=OFF is mandatory for CI-built wheels
`vendor/llama.cpp/ggml/CMakeLists.txt` defaults `GGML_NATIVE=ON` for non-cross-compile builds. With `GGML_NATIVE=ON`, `ggml-cpu/CMakeLists.txt` runs `FindSIMD.cmake` on MSVC and sets `GGML_AVX512=ON` if the build host has AVX-512, baking `/arch:AVX512` into every ggml-cpu object. GitHub's Windows runners don't guarantee stable CPU SKUs — sometimes Azure lands an AVX-512 VM, sometimes not — so the resulting wheel is non-deterministic. With `GGML_NATIVE=OFF`, our explicit flags take effect (`GGML_AVX2=off, GGML_FMA=off, GGML_F16C=off`; `GGML_AVX=on` by default from `INS_ENB`), producing an `/arch:AVX` baseline binary that works on any x86-64 CPU from 2011 onward.

### nvcuda.dll stub via pefile auto-discovery (not a hand-written minimal set)
An earlier attempt statically listed nvcuda exports, which broke whenever llama.cpp added a new CUDA driver-API call. Final pattern: at CI time, install pefile into the venv python, scan both the CUDA toolkit's `Library\bin\*.dll` and the wheel's `llama_cpp\lib\*.dll` for `IMAGE_DIRECTORY_ENTRY_IMPORT` entries referencing `nvcuda.dll`, collect every imported function name, and emit a C file with one `__declspec(dllexport) int Foo() { return 100; }` per symbol (100 = CUDA_ERROR_NO_DEVICE). Compile with `cl /nologo /O2 /LD` via vcvarsall. Copy to `CONDA_PREFIX/Library/bin`. Stub works forever.

### pyproject.toml patched at CI time, not upstream
Three edits during build: `minimum-version = "0.5.1"` → `"0.9.2"` (exit scikit-build-core compat mode), `cmake.minimum-version = "3.21"` → `cmake.version = ">=3.21"` (rename deprecated key), inject `cmake.build-type = "Release"`. All three restored via `git checkout -- pyproject.toml` after the wheel is built. Same restore-pattern as the `__init__.py` version embed from Phase 2. Keeps the tracked file clean so merges from upstream remain painless.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] CUDA DLL path not exported to venv steps**
- **Found during:** Task 2 (first dispatch, import failed with "DLL load failed")
- **Issue:** ctypes couldn't locate cudart64_12.dll because the venv python step didn't inherit the CUDA bin path
- **Fix:** Export `PATH` and `CUDA_PATH` via `$env:GITHUB_ENV` so downstream venv steps see the CUDA toolkit bin directory
- **Committed in:** `44ed106, 5bd3394, bcfb503`

**2. [Rule 1 - Bug] nvcuda.dll absent on GitHub runners**
- **Found during:** Task 2 (dispatch, ggml-cuda.dll load chain broken at nvcuda.dll)
- **Issue:** GitHub-hosted Windows runners have no NVIDIA GPU driver, so nvcuda.dll is absent. ggml-cuda.dll's DllMain fails to resolve nvcuda driver-API functions → DLL load failure
- **Fix:** CI-time pefile-discovered nvcuda.dll stub (5 successive iterations to make the discovery complete: scan CUDA toolkit DLLs, scan wheel DLLs, make stub exports include every discovered function)
- **Committed in:** `202a70b, 7edede5, 17d3128, 1b10a13, 0ccf11f, 2b96812, 79d2c0e`

**3. [Rule 1 - Bug] Debug CRT linked, Debug codegen — 0xe06d7363 C++ exception**
- **Found during:** Task 2 (dispatch, inference test threw C++ exception 0xe06d7363)
- **Issue:** scikit-build-core defaults to `CMAKE_BUILD_TYPE=Debug` and silently ignores every override channel. Debug CRT (`MSVCP140D.dll, ucrtbased.dll`) was linked, and Debug compile flags (`/Od /RTC1`) were applied, causing C++ exceptions during `llama_tokenize`
- **Fix:** After 7 failed override attempts, forced Release CRT via `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` + `CMP0091=NEW`, and stripped `/Od /RTC1` via `CMAKE_{C,CXX,CUDA}_FLAGS_DEBUG="/O2 /Ob2 /DNDEBUG"` overrides
- **Committed in:** `cfbe73b, 8677982, 98590db, 9eddb01, 0830df7, 08500ae, b34a230, c3e48b8`

**4. [Rule 1 - Bug] tiny-llama.gguf fixture triggered BPE edge case**
- **Found during:** Task 2 (dispatch with Release codegen, inference still threw 0xe06d7363)
- **Issue:** The committed 27 KB fixture had `n_vocab=32, n_layer=1, n_head=1, n_embd=32` — pathologically degenerate. BPE/SPM tokenizer code path assumed realistic vocab sizes and threw on the tiny vocab
- **Fix:** Swap fixture to `lmstudio-community/Qwen3.5-0.8B-Q8_0.gguf` via HuggingFace Hub download with actions/cache
- **Committed in:** `6a79267`

**5. [Rule 1 - Bug] STATUS_ILLEGAL_INSTRUCTION from /arch:AVX512 on non-AVX-512 smoke-test runner**
- **Found during:** Task 2 (dispatch with Qwen fixture, smoke-test crashed with 0xc000001d during inference)
- **Issue:** `GGML_NATIVE=ON` (the default) activated `FindSIMD.cmake` on the build host. The build runner had AVX-512 → `GGML_AVX512=ON` set → `/arch:AVX512` baked into ggml-cpu.dll → illegal-instruction on non-AVX-512 smoke-test runner
- **Fix:** `-DGGML_NATIVE=OFF` so explicit GGML_AVX*/GGML_FMA/GGML_F16C flags take effect, producing `/arch:AVX` baseline binary
- **Committed in:** `5f39e6a`

**6. [Rule 3 - Scope call] CPU inference test has negative value for GPU users; simplified to DLL-load gate**
- **Found during:** Task 2 (after landing GGML_NATIVE=OFF, user pointed out their real workload uses `n_gpu_layers=-1` which never exercises the CPU path)
- **Issue:** CPU-only inference on CI surfaced failures (fixture edges, arch mismatches) that never hit end users. Plan's ST-06 was gating on something users don't use
- **Fix:** Simplified to DLL-load import gate (`from llama_cpp import Llama, llama_cpp`). Still catches upstream #1543 (wheel compiles but crashes on import) without the CPU-path false-fail surface. Removed HF cache / GGUF download / inference steps. Deleted tiny-llama.gguf fixture
- **User approved:** Yes, in session
- **Committed in:** `50fa0d9`

---

**Total deviations:** 6 (5 Rule 1 bugs auto-fixed, 1 Rule 3 scope call approved by user)

**Impact on plan:** All Rule 1 fixes were discovered during dispatch verification (Task 2's explicit purpose). The Rule 3 scope call (simplifying inference → DLL-load gate) is a plan deviation approved by the user after empirical demonstration that the original ST-06 gate was testing the wrong thing for this fork's user base. Two plan requirements are superseded (ST-01 fixture, ST-06 inference) but the phase goal — a hard publish gate preventing segfaulting wheels from shipping — is preserved and even sharpened (the DLL-load gate has zero false-fail surface for CPU hot paths users don't hit).

## Issues Encountered

- **Extensive dispatch iteration count.** 16+ fix commits across 2 days. Root cause: scikit-build-core aggressively ignores every override channel for `CMAKE_BUILD_TYPE`, which triggered 7 successive attempts before the CMAKE_MSVC_RUNTIME_LIBRARY + CMAKE_*_FLAGS_DEBUG workaround was discovered. Combined with the CPU-path false-fail compound from the bad fixture and non-deterministic runner CPU features, the debug loop was longer than anticipated.
- **Non-determinism from GitHub-hosted Windows runner CPU SKUs.** Wheels built yesterday may have different `/arch:` flags than wheels built today, depending on which Azure VM the runner lands on. `GGML_NATIVE=OFF` fully fixes this — all wheels now produce `/arch:AVX` baseline — but it's a trap any CI for llama.cpp-derived builds needs to avoid.
- **Downstream users on Zen 3+ GPUs.** User confirmed via `llm_form_analyser_qwen.py` that their real workload is `n_gpu_layers=-1` on an RTX 4090, which sidesteps CPU-path bugs. This was the insight that motivated the simplification to DLL-load gate.

## Authentication Gates

None — no external service authentication required.

## User Setup Required

None — no external service configuration required. The HF model cache step added in `6a79267` was removed in `50fa0d9` along with the inference test.

## Next Phase Readiness

- **Phase 3 COMPLETE.** Smoke-test job exists, runs on fresh `windows-2022` runner, declares `needs: [build]`, passes all assertions on a real dispatch (run `24611962164`).
- **Ready for Phase 4 (Publish Gate):** Phase 4's publish job will declare `needs: [build, smoke-test]` and add a lint-workflow grep-assertion verifying that dependency exists. The smoke-test job's existence, ID, and contract are stable.
- **Requirements satisfied:** ST-02, ST-03, ST-04, ST-05, ST-07 all verified green.
- **Requirements superseded:** ST-01 (fixture), ST-06 (inference). Both replaced by the DLL-load import gate with user approval.
- **Requirements deferred:** ST-08 (publish job `needs: [build, smoke-test]` gate) — Phase 4 scope.
- **Portable wheels guaranteed.** `GGML_NATIVE=OFF` ensures every future build produces `/arch:AVX` baseline binary runnable on any x86-64 CPU from 2011+.
- **No blockers.** Phase 4 can begin immediately.

---

## Self-Check: PASSED

- `.github/workflows/build-wheels-cuda-windows.yaml` contains `smoke-test:` job with `needs: [build]` and `runs-on: windows-2022`
- `llm_for_ci_test/` directory removed; tiny-llama.gguf no longer tracked
- All 16+ fix commits pushed to main
- Run `24611962164` shows all three jobs green (build + smoke-test + lint-workflow)
- Wheel `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` (340 MB) verified loadable on a driver-less runner via DLL-load import gate

---

*Phase: 03-smoke-test-publish-gate*
*Completed: 2026-04-18*
