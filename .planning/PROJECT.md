# Windows CUDA Wheels CI for llama-cpp-python (fork)

## What This Is

A GitHub Actions workflow (`.github/workflows/build-wheels-cuda-windows.yaml`), living in this fork of `abetlen/llama-cpp-python`, that builds and publishes runtime-verified Windows x64 CUDA wheels of `llama-cpp-python` on `workflow_dispatch`. Consumers install by downloading the `.whl` from the [Releases page](https://github.com/ThongvanAlexis/llama-cpp-python/releases) and running `pip install path/to/wheel.whl`. The upstream maintainer stopped publishing Windows CUDA wheels in July 2025 (commit `98fda8c`, "Temporarily" — still off), leaving Windows users stuck at 0.3.4. This fork reactivates that path.

## Current State

**Shipped:** v1.0 (2026-04-19) — live Release at [`v0.3.20-cu126-win`](https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v0.3.20-cu126-win) with runtime-verified wheel (`llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl`, 356 MB).

Summary: 4 phases / 8 plans over ~5 days; green end-to-end dispatch `24623672816`; workflow spans 1842 lines; four-job DAG (build → smoke-test → publish, lint-workflow parallel) with structural publish gate; all 42 in-scope v1 requirements satisfied or user-approved superseded (2 supersessions, 1 cosmetic partial — see `.planning/milestones/v1.0-MILESTONE-AUDIT.md`).

## Core Value

**Produce a Windows x64 CUDA wheel that actually works at runtime** (loads a model, runs inference without segfault) and is installable by downloading the `.whl` from the [Releases page](https://github.com/ThongvanAlexis/llama-cpp-python/releases) and running `pip install path/to/wheel.whl`.

The runtime-works part is non-negotiable: upstream's historical failure mode is wheels that install fine but segfault on first `Llama(...)` call when `-allow-unsupported-compiler` was used to paper over VS/nvcc incompatibility. Every publish is gated on a passing smoke test.

## Requirements

### Validated

<!-- Inferred from existing codebase — the llama-cpp-python library itself already works. -->

- ✓ Python bindings to llama.cpp C++ library (ctypes + vendored llama.cpp submodule) — upstream
- ✓ CMake + scikit-build-core build system producing platform wheels — upstream
- ✓ Existing cross-platform CI workflow `.github/workflows/build-wheels-cuda.yaml` with Windows steps already written but commented out — upstream
- ✓ Existing test suite under `tests/` (pytest) — upstream

<!-- Shipped v1.0 — the new scope this fork delivered. -->

- ✓ New workflow file `.github/workflows/build-wheels-cuda-windows.yaml` — v1.0
- ✓ `workflow_dispatch` inputs for `python_version` (default 3.11) and `cuda_version` (default 12.6.3, bumped from 12.4.1 after OQ1) and `msvc_toolset` (default `auto`) — v1.0
- ✓ MSVC auto-select from CUDA compat matrix (resilient to runner image rotation) — v1.0
- ✓ Single CUDA install path via mamba `cuda-toolkit`; no dual-install via Jimver — v1.0
- ✓ sccache wired into CMake as C / CXX / CUDA compiler launcher; 5 GB cache, `append-timestamp: false` — v1.0
- ✓ Mamba pkgs cache with split restore/save + `if: always()` save-on-failure — v1.0
- ✓ `-allow-unsupported-compiler` hard-banned via regex-char-class grep-assert in lint-workflow — v1.0
- ✓ Correctly-tagged `cp311-cp311-win_amd64` wheel under 400 MB with `+cu126.ll<sha12>` version embed; CI asserts tag and size — v1.0
- ✓ Smoke-test publish gate (`needs: [build, smoke-test]`) via DLL-load import gate + cuobjdump arch check (sm_80/86/89/90) on driverless runner — v1.0 (ST-01 fixture + ST-06 CPU-inference superseded by user-approved DLL-load gate)
- ✓ Portable `/arch:AVX` wheels via `GGML_NATIVE=OFF` — v1.0
- ✓ Release CRT enforcement via `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` + `CMAKE_*_FLAGS_DEBUG` overrides — v1.0
- ✓ nvcuda.dll stub via pefile auto-discovery for driverless runner — v1.0
- ✓ Publish job on `ubuntu-latest` with `softprops/action-gh-release@v2`, byte-identical upload, forensics body embedding 12-char llama.cpp SHA — v1.0
- ✓ Fork-focused `README.md` with manual-download install flow + NVIDIA driver floor (≥ 561.17); upstream README preserved at `README.upstream.md` — v1.0

### Active

<!-- v2 scope — nothing scheduled yet. Next milestone to be defined via /gsd:new-milestone. -->

*(None — planning next milestone. v2 candidates below are ideas not yet accepted.)*

### v2 Candidates (deferred — not in Active until a v2 milestone is started)

- Matrix expansion: multi-Python (3.10/3.12) and multi-CUDA (12.4/12.8) once the single-combo is stable for ~1 month (MX-01, MX-02)
- Python 3.13 / free-threaded builds (MX-03)
- Auto-trigger on tag push / Release publish; weekly canary; upstream-drift checker; dependabot (AUT-01..05)
- Sigstore / PEP 740 attestations; SBOM; PEP 658 metadata; `delvewheel repair` (QA-01..04)
- gh-pages PEP 503 pip index (PUB-03..PUB-10 + DOC-03) — deferred 2026-04-19; revisit only if fork user base grows

### Out of Scope

- **Linux wheels** — upstream still publishes these; no reason to duplicate
- **macOS / Metal wheels** — upstream still publishes these
- **PyPI publishing** — name collision with upstream; we intentionally ship via GitHub Releases only
- **Upstream PR to reactivate Windows in abetlen/llama-cpp-python** — upstream is in maintenance pause (#2136 Quansight offer ignored)
- **`-allow-unsupported-compiler` fallback** — banned. Produces wheels that segfault at runtime (upstream #1543)
- **CPU-only Windows wheels** — already exist via `pip install llama-cpp-python`; scope creep
- **GPG signing** — dead standard replaced by sigstore; no consumer demand for this fork
- **Auto-PR to upstream** — maintainer non-responsive; wasted effort
- **jllllll-style full-matrix default** — 4×4×N combinatorics killed jllllll's fork; we stay narrow

## Context

**Why this exists (user spec, `.planning/user_specs/001_WINDOWS_CUDA_WHEELS_CI_PLAN.md`):**
- Upstream abandoned Windows CUDA wheels in July 2025 after a runner-image churn week.
- The community fallback index `jllllll.github.io/llama-cpp-python-cuBLAS-wheels` has been frozen at 0.2.26 since Jan 2024.
- Downstream consumers (APHP project using this fork) need modern llama.cpp features (Qwen3, Gemma 3, flash attention) on Windows + CUDA.

**Tech stack (v1.0 as shipped):**
- GitHub Actions YAML only — no custom runners, no external shell scripts.
- `windows-2022` for build + smoke-test; `ubuntu-latest` for publish (5× cheaper minutes).
- scikit-build-core build; Ninja generator; sccache + mamba-pkgs split caches.
- Targets Python 3.11 + CUDA 12.6.3 by default (dispatch inputs allow override).
- Wheel size: ~356 MB (under the 400 MB guard).
- Cold build: ~120 min; warm build meaningfully shorter with >30% sccache hits.

**Known issues / tech debt carried into v2:**
- Phase 3 VERIFICATION.md never written (evidence in dispatch runs; see v1.0-MILESTONE-AUDIT.md).
- README.md:76 has stale tiny-llama-GGUF description — cosmetic; consumer install flow unaffected.
- Phase 2 + Phase 3 VALIDATION.md still `draft` / `nyquist_compliant: false`.
- Orphan step output `build.outputs.selected_full_version` — surfaced but unused in publish body.

## Constraints

- **Platform**: Windows x64 runner (`windows-2022`). Not chasing `windows-2025` until VS / nvcc alignment is empirically confirmed.
- **Python target (v1)**: 3.11 default, 3.10/3.12 selectable via dispatch input. 3.13 / free-threaded deferred to v2.
- **CUDA target (v1)**: 12.6.3 default; bumped from 12.4.1 on 2026-04-15 after OQ1 (VC.14.39.17.9 component retired from windows-2022 image per actions/runner-images#9701).
- **Build budget**: ≤ 6h per job (GitHub free-tier runner timeout). Actual: ~120 min cold, meaningfully shorter warm.
- **Wheel size**: under 400 MB (hard CI assert). Actual: ~356 MB.
- **No reliance on upstream merge**: workflow lives in a separate file (`build-wheels-cuda-windows.yaml`) so rebases from upstream don't stomp it.
- **Smoke test gates publish**: red smoke-test is structurally incapable of reaching the Release API.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| New file `build-wheels-cuda-windows.yaml`, not edit existing | Avoid upstream merge conflicts | ✓ Good (v1.0) |
| `workflow_dispatch`-only trigger in v1 | Minimize noise while stabilizing VS/nvcc fight; tag/release triggers deferred to v2 | ✓ Good (v1.0) |
| Dispatch inputs for `python_version` + `cuda_version`; `msvc_toolset=auto` | Lets us rebuild for different combos without editing YAML | ✓ Good (v1.0) |
| Bump default from CUDA 12.4.1 → 12.6.3 and MSVC 14.39 → 14.40 (OQ1 resolution) | `VC.14.39.17.9` component removed from windows-2022 image (actions/runner-images#9701). 14.40 requires nvcc `_MSC_VER` cap ≥ 1940, starts at CUDA 12.5; 12.6.3 confirmed available on `nvidia/label/cuda-12.6.3` channel. | ✓ Good (2026-04-15) |
| MSVC auto-select from CUDA compat matrix | Resilient to runner image rotation. Probe enumerates `VC\Tools\MSVC\*` from disk, looks up CUDA `_MSC_VER` cap (CUDA 12.4–12.9 all cap at < 1950), picks newest compatible toolset. | ✓ Good (2026-04-16) |
| Ban `-allow-unsupported-compiler` via regex-char-class grep | Historical failure mode: compiles but segfaults at runtime (upstream #1543). Char class `[-]` avoids self-match bug. | ✓ Good (v1.0) |
| Drop `ilammy/msvc-dev-cmd`; call `vcvarsall.bat` in clean pwsh subprocess | Conda + ilammy both call `vcvarsall.bat` → overflows CMD's 8192-char INCLUDE/LIB limit. Subprocess approach isolates env diff. | ✓ Good (2026-04-16) |
| sccache `append-timestamp: false`, `max-size: 5G`, `CMAKE_BUILD_PARALLEL_LEVEL=1` | Stable cache keys across runs; parallel nvcc killed sccache daemon (OS 10054). | ✓ Good (2026-04-17) |
| `-C wheel.py-api=""` to override `pyproject.toml wheel.py-api="py3"` | `SKBUILD_WHEEL_PY_API=''` env var doesn't work (empty strings ignored). Produces cp311-cp311 tag, not py3-none. | ✓ Good (2026-04-17) |
| Smoke test simplified from CPU inference to DLL-load import gate | CPU hot paths are never exercised by `n_gpu_layers=-1` users this fork serves; GGUF-fixture false-fails and `/arch:AVX512` illegal-instruction crashes were chasing the wrong failure mode. DLL-load still catches upstream #1543. | ✓ Good (2026-04-18, user-approved) |
| nvcuda.dll stub via pefile auto-discovery (not hand-written) | Runners have no NVIDIA driver. Stub discovers exports from both CUDA toolkit + wheel DLLs, compiles with `cl /O2 /LD`. Forward-compatible across llama.cpp versions. | ✓ Good (2026-04-18) |
| `GGML_NATIVE=OFF` for portability | `FindSIMD.cmake` on MSVC with `GGML_NATIVE=ON` bakes `/arch:AVX512` whenever the build runner has AVX-512. Azure VM SKU rotation made wheels non-deterministic. OFF forces `/arch:AVX` baseline. | ✓ Good (2026-04-18) |
| Release CRT via `CMAKE_MSVC_RUNTIME_LIBRARY` + `CMAKE_*_FLAGS_DEBUG` override | scikit-build-core strips every documented `CMAKE_BUILD_TYPE=Release` channel. Direct CMake variable assignments ARE honored. | ✓ Good (2026-04-18) |
| 2026-04-19: Drop gh-pages PEP 503 index (Release-only publishing) | Maintenance cost of gh-pages branch + index regen + Fastly probe + `--extra-index-url` UX didn't justify complexity for a solo-maintainer fork. PUB-03..10 + DOC-03 moved to Out of Scope. | ✓ Good (v1.0) |
| Parse 12-char llama.cpp SHA from wheel filename via `sed` in publish job | Avoids a new `assert-submodule` step output and keeps Phase 1 untouched. | ✓ Good (v1.0) |
| CUDA-runtime-bundling disclosure in release body | User visual inspection revealed consumers might incorrectly infer they need CUDA Toolkit 12.6.3 locally. Two-path remediation: `gh release edit` on live body + workflow template update. | ✓ Good (2026-04-19) |

---
*Last updated: 2026-04-19 after v1.0 milestone close.*
