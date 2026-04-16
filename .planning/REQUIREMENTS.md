# Requirements: Windows CUDA Wheels CI for llama-cpp-python (fork)

**Defined:** 2026-04-15
**Core Value:** Produce a Windows x64 CUDA wheel that actually works at runtime (loads a model and runs inference without segfault) and is installable via `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu126`.

## v1 Requirements

### Workflow Scaffolding

- [x] **WF-01**: New workflow file `.github/workflows/build-wheels-cuda-windows.yaml` exists, separate from upstream's `build-wheels-cuda.yaml` (avoid merge conflicts on upstream pulls)
- [x] **WF-02**: Workflow is triggered only by `workflow_dispatch` (no push/tag/release auto-triggers in v1)
- [x] **WF-03**: Workflow exposes `python_version` input (default `3.11`) selectable at dispatch time
- [x] **WF-04**: Workflow exposes `cuda_version` input (default `12.6.3`; originally spec'd as `12.4.1`, bumped 2026-04-15 after OQ1 resolution — see STATE.md Decisions) selectable at dispatch time
- [x] **WF-05**: Workflow checks out the repo with `submodules: recursive` so `vendor/llama.cpp/` is populated

### Toolchain Pinning (Load-Bearing)

- [x] **TC-01**: Runner is pinned to `windows-2022` explicitly (never `windows-latest`)
- [x] **TC-02**: MSVC toolset is auto-selected from runner by enumerating `VC\Tools\MSVC\*` directories, looking up the CUDA _MSC_VER cap from a compatibility matrix (host_config.h), and picking the newest compatible toolset. msvc_toolset input defaults to 'auto'; override mode validates both installed AND CUDA-compatible. Originally spec'd as vs_installer install (abandoned 2026-04-16); then enumerate+pin (abandoned 2026-04-16); now auto-select from compat matrix (2026-04-16).
- [x] **TC-03**: MSVC toolset is verified via cl.exe called by full path (`$installPath\VC\Tools\MSVC\$selectedFull\bin\HostX64\x64\cl.exe`) — `ilammy/msvc-dev-cmd` was dropped because conda activation + ilammy both call vcvarsall.bat, overflowing CMD's 8192-char INCLUDE/LIB limit. Phase 2 build will use cmake's VS generator or a clean vcvarsall subprocess for full VS environment activation
- [x] **TC-04**: Preflight step asserts two things: (1) pin integrity — cl.exe _MSC_VER matches expected value for selected toolset, (2) CUDA compatibility — _MSC_VER ≤ cap from compat matrix. Both checks use probe-msvc step outputs, not raw input.
- [x] **TC-05**: CUDA toolkit is installed via a single path (mamba `cuda-toolkit`); no parallel Jimver full-installer install
- [x] **TC-06**: Preflight step asserts `nvcc --version` matches the requested `cuda_version` input
- [x] **TC-07**: `CUDA_PATH` and `CUDA_PATH_V*_*` env variables are explicitly unset or normalized before build (prevent double-install confusion)
- [x] **TC-08**: Visual Studio BuildCustomizations path is detected dynamically (no hardcoded `2019\Enterprise` path from upstream)
- [x] **TC-09**: Windows `LongPathsEnabled=1` registry key is set before checkout
- [x] **TC-10**: CMake flag `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` is **not** used anywhere; CI fails if the workflow contains that literal (grep-assert)

### Build + Caching

- [ ] **BLD-01**: Wheel is produced via `python -m build --wheel` using scikit-build-core (not cibuildwheel)
- [ ] **BLD-02**: sccache is set up via `hendrikmuhs/ccache-action@v1.2` with `variant: sccache`
- [ ] **BLD-03**: sccache is wired into CMake via `-DCMAKE_C_COMPILER_LAUNCHER=sccache`, `-DCMAKE_CXX_COMPILER_LAUNCHER=sccache`, `-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache`
- [ ] **BLD-04**: MSVC debug info is forced to `/Z7` (embedded) via `CMAKE_POLICY_DEFAULT_CMP0141=NEW` + `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded` so sccache can hash TU output deterministically
- [ ] **BLD-05**: CUDA installer zip is cached with `actions/cache@v4` keyed on `cuda-installer-${cuda_version}-windows`, saved `if: always()` (survives build failure)
- [ ] **BLD-06**: Mamba/conda package directory is cached with `actions/cache@v4` keyed on `mamba-cuda-${cuda_version}-${runner.os}` with `restore-keys` fallback, saved `if: always()`
- [ ] **BLD-07**: VS CUDA integration extensions are cached (key `cuda-${cuda_version}-vs-integration`), saved `if: always()`
- [ ] **BLD-08**: Build uses Ninja generator with `CMAKE_BUILD_PARALLEL_LEVEL` set appropriately for the runner
- [ ] **BLD-09**: Each job step has a `timeout-minutes` bound to prevent 6h runner-timeout surprises
- [ ] **BLD-10**: Produced wheel is tagged correctly: `llama_cpp_python-<version>-cp<pyver>-cp<pyver>-win_amd64.whl` (never `abi3` or `none-any`); wheel-tag regex assertion guards against drift
- [ ] **BLD-11**: Wheel size is under 400 MB; CI fails if the produced wheel exceeds that
- [ ] **BLD-12**: Wheel version embeds the llama.cpp submodule short SHA (e.g., `0.3.20+cu126.ll<sha>`) for reproducibility
- [ ] **BLD-13**: Wheel is uploaded as a GitHub Actions artifact via `actions/upload-artifact@v4` for downstream job consumption

### Smoke Test (Publish Gate)

- [ ] **ST-01**: `llm_for_ci_test/tiny-llama.gguf` (27 KB) is committed into the repo
- [ ] **ST-02**: A separate `smoke-test` job runs on a **fresh** `windows-2022` runner (not the build runner)
- [ ] **ST-03**: Smoke-test job uses sparse checkout that omits `llama_cpp/` source (so `import llama_cpp` resolves against `site-packages`, not the repo)
- [ ] **ST-04**: Smoke-test job downloads the wheel artifact from the build job via `actions/download-artifact@v4`
- [ ] **ST-05**: Smoke-test job creates a clean venv (no pre-existing llama-cpp-python install) and `pip install`s the downloaded wheel
- [ ] **ST-06**: Smoke-test runs `from llama_cpp import Llama; llm = Llama(model_path='llm_for_ci_test/tiny-llama.gguf', n_ctx=64); llm('hi', max_tokens=2)` and asserts no non-zero exit / no segfault
- [ ] **ST-07**: Smoke-test uses `cuobjdump --list-elf` on the shipped `ggml-cuda.dll` to verify expected CUDA architectures are present (compensates for CPU-only runner)
- [ ] **ST-08**: `publish` job has `needs: [build, smoke-test]` — the job DAG makes the smoke test a hard publish gate

### Publish (GitHub Pages Pip Index)

- [ ] **PUB-01**: `publish` job runs on `ubuntu-latest` (no CUDA needed for HTML generation + git push; 5× cheaper minutes)
- [ ] **PUB-02**: Wheel is uploaded as an asset on a GitHub Release (authoritative storage) via `softprops/action-gh-release@v2`
- [ ] **PUB-03**: Wheel file is placed at `whl/cu<tag>/llama-cpp-python/<wheel-filename>.whl` in the `gh-pages` branch (tag derived from CUDA major.minor, e.g., `cu126`)
- [ ] **PUB-04**: `index.html` is regenerated from the actual directory listing of `whl/cu<tag>/llama-cpp-python/` on every publish (idempotent; append-only semantics)
- [ ] **PUB-05**: `index.html` uses PEP 503-normalized project folder name `llama-cpp-python` (hyphens, lowercase)
- [ ] **PUB-06**: Each `<a>` entry in `index.html` includes a `#sha256=<digest>` URL fragment for pip hash checking
- [ ] **PUB-07**: Each `<a>` entry includes `data-requires-python` attribute derived from the wheel's Python tag
- [ ] **PUB-08**: `peaceiris/actions-gh-pages@v4` is configured with `keep_files: true` so multiple wheels across versions coexist on the index
- [ ] **PUB-09**: A `concurrency: group: gh-pages-publish, cancel-in-progress: false` block on the publish job prevents simultaneous-dispatch races
- [ ] **PUB-10**: Post-publish step probes the published URL (up to 15 × 60s) and confirms the new wheel is resolvable before the job exits green

### Documentation

- [ ] **DOC-01**: README has an "Install (Windows CUDA)" section with the canonical command: `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu126`
- [ ] **DOC-02**: README notes the minimum NVIDIA driver version (≥ 560.x for CUDA 12.6 per NVIDIA's driver-compat matrix; bumped from ≥ 551.61/CUDA 12.4 on 2026-04-15 after OQ1)
- [ ] **DOC-03**: README notes the up-to-15-minute Fastly cache delay after a publish and suggests `pip install --no-cache-dir` when a fresh wheel isn't resolved
- [x] **DOC-04**: The workflow YAML contains inline comments explaining the MSVC 14.40 pin rationale (and the 14.39 → 14.40 bump history) and the `-allow-unsupported-compiler` ban (link to upstream #1543)
- [ ] **DOC-05**: Release notes (GitHub Release body) record the llama.cpp submodule SHA at build time

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Matrix Expansion

- **MX-01**: Multi-Python matrix (3.10 / 3.11 / 3.12) — add after ~1 month of single-combo stability
- **MX-02**: Multi-CUDA matrix (12.4 / 12.6) — bounded by VS compatibility wall; requires 12.6 VS range research
- **MX-03**: Python 3.13 / free-threaded builds — depends on scikit-build-core / llama.cpp support

### Automation

- **AUT-01**: Auto-trigger on push of a version tag (workflow_dispatch supplemented, not replaced)
- **AUT-02**: Auto-trigger on GitHub Release publish
- **AUT-03**: Weekly scheduled canary build (cron) that builds current HEAD without publishing to catch image drift early
- **AUT-04**: Upstream-drift checker that compares our workflow against upstream's `build-wheels-cuda.yaml` monthly and opens an issue
- **AUT-05**: `dependabot.yml` entry for `package-ecosystem: github-actions`

### Provenance / Quality

- **QA-01**: Sigstore / PEP 740 attestations on published wheels
- **QA-02**: SBOM generation (CycloneDX) alongside wheel
- **QA-03**: PEP 658 detached metadata `.metadata` files served alongside wheels
- **QA-04**: `delvewheel repair` to vendor non-CUDA transitive DLLs (VC++ runtime, etc.) into the wheel

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Linux wheels | Upstream still publishes these; no reason to duplicate |
| macOS / Metal wheels | Upstream still publishes these |
| PyPI publishing | Name collision with upstream; we intentionally ship via gh-pages index only |
| Upstream PR to reactivate Windows in abetlen/llama-cpp-python | Upstream is in maintenance pause (#2136 Quansight offer ignored); not worth the effort |
| `-allow-unsupported-compiler` fallback | Produces wheels that segfault at runtime (upstream #1543) |
| CPU-only Windows wheels | Already exist via `pip install llama-cpp-python`; scope creep |
| GPG signing | Dead standard replaced by sigstore; no consumer demand for this fork |
| Elaborate UI / dashboard | pip doesn't consume a UI |
| Mirror to multiple registries | One gh-pages index is enough for our downstream |
| jllllll-style full-matrix default | 4×4×N combinatorics killed jllllll's fork; we stay narrow and widen on demand |
| Auto-PR to upstream | Maintainer non-responsive; wasted effort |
| Python 3.13 / free-threaded in v1 | Upstream hasn't solved it (#2136); not our immediate need |

## Traceability

Which phases cover which requirements. Populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| WF-01 | Phase 1 | Complete |
| WF-02 | Phase 1 | Complete |
| WF-03 | Phase 1 | Complete |
| WF-04 | Phase 1 | Complete |
| WF-05 | Phase 1 | Complete |
| TC-01 | Phase 1 | Complete |
| TC-02 | Phase 1 | Complete |
| TC-03 | Phase 1 | Complete |
| TC-04 | Phase 1 | Complete |
| TC-05 | Phase 1 | Complete |
| TC-06 | Phase 1 | Complete |
| TC-07 | Phase 1 | Complete |
| TC-08 | Phase 1 | Complete |
| TC-09 | Phase 1 | Complete |
| TC-10 | Phase 1 | Complete |
| BLD-01 | Phase 2 | Pending |
| BLD-02 | Phase 2 | Pending |
| BLD-03 | Phase 2 | Pending |
| BLD-04 | Phase 2 | Pending |
| BLD-05 | Phase 2 | Pending |
| BLD-06 | Phase 2 | Pending |
| BLD-07 | Phase 2 | Pending |
| BLD-08 | Phase 2 | Pending |
| BLD-09 | Phase 2 | Pending |
| BLD-10 | Phase 2 | Pending |
| BLD-11 | Phase 2 | Pending |
| BLD-12 | Phase 2 | Pending |
| BLD-13 | Phase 2 | Pending |
| ST-01 | Phase 3 | Pending |
| ST-02 | Phase 3 | Pending |
| ST-03 | Phase 3 | Pending |
| ST-04 | Phase 3 | Pending |
| ST-05 | Phase 3 | Pending |
| ST-06 | Phase 3 | Pending |
| ST-07 | Phase 3 | Pending |
| ST-08 | Phase 3 | Pending |
| PUB-01 | Phase 4 | Pending |
| PUB-02 | Phase 4 | Pending |
| PUB-03 | Phase 4 | Pending |
| PUB-04 | Phase 4 | Pending |
| PUB-05 | Phase 4 | Pending |
| PUB-06 | Phase 4 | Pending |
| PUB-07 | Phase 4 | Pending |
| PUB-08 | Phase 4 | Pending |
| PUB-09 | Phase 4 | Pending |
| PUB-10 | Phase 4 | Pending |
| DOC-01 | Phase 4 | Pending |
| DOC-02 | Phase 4 | Pending |
| DOC-03 | Phase 4 | Pending |
| DOC-04 | Phase 1 | Complete |
| DOC-05 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 51 total
- Mapped to phases: 51 ✓
- Unmapped: 0

**Phase distribution:**
- Phase 1 (Scaffold & Toolchain Pinning): 16 requirements (WF-01..05, TC-01..10, DOC-04)
- Phase 2 (Build & Cache): 13 requirements (BLD-01..13)
- Phase 3 (Smoke Test): 8 requirements (ST-01..08)
- Phase 4 (Publish & Consumer UX): 14 requirements (PUB-01..10, DOC-01, DOC-02, DOC-03, DOC-05)

---
*Requirements defined: 2026-04-15*
*Last updated: 2026-04-15 — traceability populated after roadmap creation; OQ1 resolution (later the same day) bumped the cuda_version/msvc_toolset defaults from 12.4.1/14.39 to 12.6.3/14.40 and amended TC-02/TC-03/TC-04, WF-04, BLD-12, PUB-03, DOC-01/02/04, plus core-value URL (cu124 → cu126).*
