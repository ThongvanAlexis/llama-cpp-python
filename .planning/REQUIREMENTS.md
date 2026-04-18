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

- [x] **BLD-01**: Wheel is produced via `python -m build --wheel` using scikit-build-core (not cibuildwheel)
- [x] **BLD-02**: sccache is set up via `hendrikmuhs/ccache-action@v1.2` with `variant: sccache`
- [x] **BLD-03**: sccache is wired into CMake via `-DCMAKE_C_COMPILER_LAUNCHER=sccache`, `-DCMAKE_CXX_COMPILER_LAUNCHER=sccache`, `-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache`
- [x] **BLD-04**: MSVC debug info is forced to `/Z7` (embedded) via `CMAKE_POLICY_DEFAULT_CMP0141=NEW` + `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded` so sccache can hash TU output deterministically
- [x] **BLD-05**: CUDA installer zip is cached with `actions/cache@v4` keyed on `cuda-installer-${cuda_version}-windows`, saved `if: always()` (survives build failure)
- [x] **BLD-06**: Mamba/conda package directory is cached with `actions/cache@v4` keyed on `mamba-cuda-${cuda_version}-${runner.os}` with `restore-keys` fallback, saved `if: always()`
- [x] **BLD-07**: VS CUDA integration extensions are cached (key `cuda-${cuda_version}-vs-integration`), saved `if: always()`
- [x] **BLD-08**: Build uses Ninja generator with `CMAKE_BUILD_PARALLEL_LEVEL` set appropriately for the runner
- [x] **BLD-09**: Each job step has a `timeout-minutes` bound to prevent 6h runner-timeout surprises
- [x] **BLD-10**: Produced wheel is tagged correctly: `llama_cpp_python-<version>-cp<pyver>-cp<pyver>-win_amd64.whl` (never `abi3` or `none-any`); wheel-tag regex assertion guards against drift
- [x] **BLD-11**: Wheel size is under 400 MB; CI fails if the produced wheel exceeds that
- [x] **BLD-12**: Wheel version embeds the llama.cpp submodule short SHA (e.g., `0.3.20+cu126.ll<sha>`) for reproducibility
- [x] **BLD-13**: Wheel is uploaded as a GitHub Actions artifact via `actions/upload-artifact@v4` for downstream job consumption

### Smoke Test (Publish Gate)

- [ ] **ST-01**: `llm_for_ci_test/tiny-llama.gguf` (27 KB) is committed into the repo
- [ ] **ST-02**: A separate `smoke-test` job runs on a **fresh** `windows-2022` runner (not the build runner)
- [ ] **ST-03**: Smoke-test job uses sparse checkout that omits `llama_cpp/` source (so `import llama_cpp` resolves against `site-packages`, not the repo)
- [ ] **ST-04**: Smoke-test job downloads the wheel artifact from the build job via `actions/download-artifact@v4`
- [ ] **ST-05**: Smoke-test job creates a clean venv (no pre-existing llama-cpp-python install) and `pip install`s the downloaded wheel
- [ ] **ST-06**: Smoke-test runs `from llama_cpp import Llama; llm = Llama(model_path='llm_for_ci_test/tiny-llama.gguf', n_ctx=64); llm('hi', max_tokens=2)` and asserts no non-zero exit / no segfault
- [ ] **ST-07**: Smoke-test uses `cuobjdump --list-elf` on the shipped `ggml-cuda.dll` to verify expected CUDA architectures are present (compensates for CPU-only runner)
- [ ] **ST-08**: `publish` job has `needs: [build, smoke-test]` — the job DAG makes the smoke test a hard publish gate

### Publish (GitHub Release asset)

- [x] **PUB-01**: `publish` job runs on `ubuntu-latest` (no CUDA needed for Release asset upload; 5× cheaper minutes)
- [x] **PUB-02**: Wheel is uploaded as an asset on a GitHub Release (authoritative storage) via `softprops/action-gh-release@v2`

*PUB-03..PUB-10 moved to "Out of Scope" on 2026-04-19 — see CONTEXT.md 2026-04-19 for rationale. The gh-pages PEP 503 pip index scope was dropped in favour of manual download from the Releases page. Deferred requirement set (PEP 503 path placement, index.html regen, PEP 503 normalized name, `#sha256=` fragments, `data-requires-python`, `keep_files: true`, concurrency block, post-publish Fastly probe) is preserved in the Out of Scope table for a possible v2 revisit.*

### Documentation

- [x] **DOC-01**: README `Install (Windows CUDA)` section explains the manual download + `pip install path/to/wheel.whl` flow, points at `https://github.com/ThongvanAlexis/llama-cpp-python/releases`, and lists prereqs including the NVIDIA driver floor (≥ 561.17).
- [x] **DOC-02**: README notes the minimum NVIDIA driver version (≥ 560.x for CUDA 12.6 per NVIDIA's driver-compat matrix; bumped from ≥ 551.61/CUDA 12.4 on 2026-04-15 after OQ1)

*DOC-03 moved to "Out of Scope" on 2026-04-19 — no Fastly delay without gh-pages.*

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
| PUB-03..PUB-10 gh-pages PEP 503 pip index (path placement, index.html regen, PEP 503 normalized name, `#sha256=` fragments, `data-requires-python`, `keep_files: true`, concurrency block, post-publish Fastly probe) | Consumer install via manual download from Releases page; gh-pages maintenance cost not worth it for a solo-maintainer fork. Requirement set stays defined for a possible v2 revisit; see CONTEXT.md 2026-04-19 for rationale. |
| DOC-03 Fastly cache delay note | Consumer install via manual download from Releases page; no gh-pages, no Fastly CDN in v1. Could be revisited in v2 if the gh-pages PEP 503 index ships. |
| Linux wheels | Upstream still publishes these; no reason to duplicate |
| macOS / Metal wheels | Upstream still publishes these |
| PyPI publishing | Name collision with upstream; we intentionally ship via GitHub Releases only |
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
| BLD-01 | Phase 2 | Complete |
| BLD-02 | Phase 2 | Complete |
| BLD-03 | Phase 2 | Complete |
| BLD-04 | Phase 2 | Complete |
| BLD-05 | Phase 2 | Complete |
| BLD-06 | Phase 2 | Complete |
| BLD-07 | Phase 2 | Complete |
| BLD-08 | Phase 2 | Complete |
| BLD-09 | Phase 2 | Complete |
| BLD-10 | Phase 2 | Complete |
| BLD-11 | Phase 2 | Complete |
| BLD-12 | Phase 2 | Complete |
| BLD-13 | Phase 2 | Complete |
| ST-01 | Phase 3 | Pending |
| ST-02 | Phase 3 | Pending |
| ST-03 | Phase 3 | Pending |
| ST-04 | Phase 3 | Pending |
| ST-05 | Phase 3 | Pending |
| ST-06 | Phase 3 | Pending |
| ST-07 | Phase 3 | Pending |
| ST-08 | Phase 3 | Pending |
| PUB-01 | Phase 4 | Complete |
| PUB-02 | Phase 4 | Complete |
| PUB-03 | Deferred to v2 | Out of Scope |
| PUB-04 | Deferred to v2 | Out of Scope |
| PUB-05 | Deferred to v2 | Out of Scope |
| PUB-06 | Deferred to v2 | Out of Scope |
| PUB-07 | Deferred to v2 | Out of Scope |
| PUB-08 | Deferred to v2 | Out of Scope |
| PUB-09 | Deferred to v2 | Out of Scope |
| PUB-10 | Deferred to v2 | Out of Scope |
| DOC-01 | Phase 4 | Complete |
| DOC-02 | Phase 4 | Complete |
| DOC-03 | Deferred to v2 | Out of Scope |
| DOC-04 | Phase 1 | Complete |
| DOC-05 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 51 total defined
- Active in v1 scope: 42 (Phase 1: 16, Phase 2: 13, Phase 3: 8, Phase 4: 5)
- Out of Scope (deferred to v2): 9 (PUB-03..10 + DOC-03; moved 2026-04-19, see CONTEXT.md)
- Mapped (active + out-of-scope): 51 ✓
- Unmapped: 0

**Phase distribution (active):**
- Phase 1 (Scaffold & Toolchain Pinning): 16 requirements (WF-01..05, TC-01..10, DOC-04)
- Phase 2 (Build & Cache): 13 requirements (BLD-01..13)
- Phase 3 (Smoke Test): 8 requirements (ST-01..08)
- Phase 4 (Publish & Consumer UX): 5 requirements (PUB-01, PUB-02, DOC-01, DOC-02, DOC-05)

---
*Requirements defined: 2026-04-15*
*Last updated: 2026-04-19 — Phase 4 scope reduction: moved PUB-03..PUB-10 and DOC-03 to Out of Scope (gh-pages PEP 503 index dropped in favour of manual download from Releases page; revisit in v2). Rewrote DOC-01 to the manual-download flow. Active Phase 4 requirement count: 14 → 5. See CONTEXT.md 2026-04-19 for rationale and PROJECT.md Key Decisions for the decision record.*
*Previous update: 2026-04-15 — traceability populated after roadmap creation; OQ1 resolution (later the same day) bumped the cuda_version/msvc_toolset defaults from 12.4.1/14.39 to 12.6.3/14.40 and amended TC-02/TC-03/TC-04, WF-04, BLD-12, PUB-03, DOC-01/02/04, plus core-value URL (cu124 → cu126).*
