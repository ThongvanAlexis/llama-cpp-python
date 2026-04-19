# Milestones

## v1.0 Windows CUDA Wheels CI (Shipped: 2026-04-19)

**Delivered:** A GitHub Actions workflow (`.github/workflows/build-wheels-cuda-windows.yaml`) produces runtime-verified Windows x64 CUDA wheels on `workflow_dispatch` and publishes them as GitHub Release assets. Consumers install by downloading `.whl` from the [Releases page](https://github.com/ThongvanAlexis/llama-cpp-python/releases) and running `pip install path/to/wheel.whl`.

**Phases completed:** 4 phases, 8 plans
**Git range:** `dfe478b` (Phase 1 scaffold, 2026-04-15) → `311d00b` (Phase 4 close, 2026-04-19)
**Timeline:** ~5 days
**Live artifact:** Release `v0.3.20-cu126-win` — wheel `llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl` (356 MB), dispatched from commit `973aeff` via run `24623672816` (all 4 jobs green end-to-end).

### Key Accomplishments

1. **Structural publish gate** — four-job DAG (`build` → `smoke-test` → `publish`; `lint-workflow` parallel) with `needs: [build, smoke-test]` on publish. Red smoke-test is structurally incapable of reaching the Release API. Enforced by a regex-char-class-anchored presence-grep in `lint-workflow` (count=1, self-match-safe).
2. **MSVC auto-select from CUDA compat matrix** — probe step enumerates installed `VC\Tools\MSVC\*` on the runner, looks up the CUDA `_MSC_VER` cap from a compatibility matrix (host_config.h caps: CUDA 12.4–12.9 all ≤ 1949), picks newest compatible toolset. Resilient to runner image rotation (actions/runner-images#9701 retired VC.14.39 and VC.14.40 from the channel manifest mid-project).
3. **Hard ban on `-allow-unsupported-compiler`** — ban-grep in `lint-workflow` using regex char class `allow[-]unsupported[-]compiler` to avoid self-match. Zero literal occurrences in the workflow. Prevents regression to the upstream #1543 failure mode (wheels that compile but segfault at runtime).
4. **Runtime-works proof via DLL-load import gate** — smoke-test on a fresh driverless `windows-2022` runner with sparse checkout (excludes `llama_cpp/`), clean venv, pip-installs the wheel, and runs `from llama_cpp import Llama, llama_cpp` to trigger the full DLL chain (`llama.dll → ggml.dll → ggml-cuda.dll → cudart/cublas/nvcuda`). Non-zero exit → red smoke-test → publish blocked.
5. **nvcuda.dll stub via pefile auto-discovery** — runners have no NVIDIA driver; at CI time `pefile` scans both CUDA-toolkit DLLs and the wheel's own DLLs for `IMAGE_DIRECTORY_ENTRY_IMPORT` entries referencing `nvcuda.dll`, synthesizes a C stub exporting every discovered function returning `CUDA_ERROR_NO_DEVICE (100)`, compiles with `cl /O2 /LD` via vcvarsall. Forward-compatible across llama.cpp versions.
6. **Portable wheels via GGML_NATIVE=OFF** — disables llama.cpp's `FindSIMD.cmake` auto-detection so the wheel is never accidentally baked with `/arch:AVX512` when the build VM happens to land on an AVX-512 SKU. Result: `/arch:AVX` baseline binary runs on any x86-64 CPU from 2011+ (Sandy Bridge/Jaguar).
7. **Release CRT enforcement** — scikit-build-core strips every documented `CMAKE_BUILD_TYPE=Release` override channel. Sidestepped via `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` + `CMP0091=NEW` (forces `/MD` Release CRT) + `CMAKE_{C,CXX,CUDA}_FLAGS_DEBUG="/O2 /Ob2 /DNDEBUG"` overrides (strips `/Od /RTC1` from the Debug config profile).
8. **Three-tier save-on-failure cache stack** — sccache (5 GB, `append-timestamp: false`), mamba pkgs (split restore/save with `if: always()`), per-run VS BuildCustomizations fetch. Cold build ~120 min; warm build meaningfully shorter with `>30%` sccache hit rate after a YAML-only edit.
9. **Forensics-rich Release body** — DOC-05-compliant narrative + table embedding the 12-char llama.cpp submodule SHA (parsed from the wheel filename via `sed`), CUDA-runtime-bundling disclosure, and full toolchain fingerprint. Byte-identical upload verified via SHA-256 match between publish-job log and live Release asset.
10. **Fork-focused README** — `## Install (Windows CUDA)` section with manual-download + `pip install path/to/wheel.whl` flow, NVIDIA driver floor (≥ 561.17), and upstream README preserved byte-for-byte at `README.upstream.md` for attribution.

### Scope Reduction (2026-04-19)

Phase 4 scope reduced from gh-pages PEP 503 pip index to Release-only publishing. Requirements PUB-03..PUB-10 and DOC-03 moved to Out of Scope (9 requirements deferred to v2). Rationale: maintenance cost of gh-pages branch + index regen + Fastly probe + `--extra-index-url` UX didn't justify complexity for a solo-maintainer fork. Core runtime-works guarantee is preserved via the smoke-test gate + Release asset upload.

### Known Gaps (documentation / process only — functional work delivered)

Per `.planning/milestones/v1.0-MILESTONE-AUDIT.md` (status: `gaps_found`). User accepted these as tech debt on milestone close:

- **Phase 3 VERIFICATION.md missing** — formal phase-local verification document never written. Functional evidence exists in two independently green dispatch runs (`24611962164`, `24623672816`) and in Phase 4 VERIFICATION's forensics JSON files. Blocker per strict workflow rules, not per runtime outcome.
- **DOC-01 partial (cosmetic)** — `README.md:76` still describes the smoke-test as loading "a 27 KB tiny-llama GGUF"; fixture was deleted in Phase 3 after user-approved simplification to DLL-load import gate. Consumer install flow unaffected.
- **Orphan step output** — `build.outputs.selected_full_version` surfaced at line 129 but never consumed by publish body (publish uses `selected_toolset` instead). Cosmetic.
- **Nyquist partial** — Phase 2 and Phase 3 `VALIDATION.md` files still `draft` with `nyquist_compliant: false`. Phase 1 and Phase 4 are compliant.
- **Phase 1 TC-03 gap in 01-VERIFICATION.md is stale** — `REQUIREMENTS.md:20` was updated to reflect the `cl.exe` full-path approach (ilammy dropped) after the verification report was written.

### Requirements Outcome (42 active / 9 out-of-scope)

| Phase | Count | Satisfied | Partial | Superseded (user-approved) |
|-------|-------|-----------|---------|----------------------------|
| 1 (WF/TC/DOC-04) | 16 | 16 | 0 | 0 |
| 2 (BLD-*) | 13 | 13 | 0 | 0 |
| 3 (ST-*) | 8 | 6 (incl. ST-08 via Phase 4) | 0 | 2 (ST-01 fixture, ST-06 inference → DLL-load gate) |
| 4 (PUB/DOC) | 5 | 4 | 1 (DOC-01) | 0 |
| **Active total** | **42** | **39 (+ ST-08)** | **1** | **2** |
| Out of Scope (v2) | 9 | — | — | PUB-03..10, DOC-03 |

### Archives

- `.planning/milestones/v1.0-ROADMAP.md`
- `.planning/milestones/v1.0-REQUIREMENTS.md`
- `.planning/milestones/v1.0-MILESTONE-AUDIT.md`
- `.planning/v1.0-INTEGRATION-REPORT.md` (cross-phase wiring check)

---
