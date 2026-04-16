# Phase 2: Build & Cache - Context

**Gathered:** 2026-04-16
**Status:** Ready for planning

<domain>
## Phase Boundary

The build step, invoked after Phase 1's green toolchain preflight, produces a correctly-tagged `cp311-cp311-win_amd64.whl` under 400 MB with a reproducible version string (embeds llama.cpp submodule SHA), while caches (sccache, mamba pkgs) are keyed to invalidate on source/toolchain change and save on failure so the fail-fix-retry loop doesn't re-pay cold costs. Phase 2 adds build + cache steps to the existing workflow file; smoke-test and publish are Phase 3 and Phase 4 respectively.

</domain>

<decisions>
## Implementation Decisions

### CMake generator & MSVC activation
- Use **Ninja** generator (not VS 2022 MSBuild) — faster builds, better sccache integration
- Activate MSVC via **vcvarsall.bat** called in a clean pwsh subprocess before building
- The vcvarsall invocation strategy (dedicated step vs inline) is Claude's discretion — key constraint is avoiding the 8192-char INCLUDE/LIB overflow discovered in Phase 1
- **Reuse probe-msvc step outputs** (install_path, selected_full_version) to locate vcvarsall.bat — single source of truth, no duplicate discovery
- Pass CMake arguments via **`$env:CMAKE_ARGS` environment variable** (matches upstream's pattern, visible in step logs)

### Wheel versioning & SHA embed
- Version format: **`0.3.20+cu126.ll<sha12>`** — PEP 440 local segment with CUDA tag and 12-char llama.cpp SHA
- Override mechanism: **`SETUPTOOLS_SCM_PRETEND_VERSION`** env var set before `python -m build` — no file modifications needed
- Base version: **read from `llama_cpp/__init__.py`** at build time using the same regex scikit-build-core uses — stays in sync with upstream version bumps automatically
- SHA length: **12 characters** (e.g. `lla1b2c3d4e5f6`) — collision-safe for the large llama.cpp repo

### Cache key strategy
- **Tight primary keys, broad restore-keys fallback** — exact match preferred, stale-but-usable cache on partial match
- sccache key includes: cuda_version, msvc_toolset, hashFiles('vendor/llama.cpp/CMakeLists.txt', 'vendor/llama.cpp/src/**') — skip .git to avoid the file-vs-directory submodule issue noted in STATE.md
- sccache restore-keys: fall back to cuda_version + msvc_toolset only (partial hit still saves recompilation)
- **BLD-05 reinterpreted**: since TC-05 locked CUDA install to mamba-only (no Jimver zip), BLD-05's intent (don't re-download CUDA on retry) is fulfilled by BLD-06 (mamba pkgs cache). No dead "CUDA installer zip" cache
- **BLD-07 (VS integration)**: skip caching — the BuildCustomizations copy takes <5 seconds; just redo it each time. Implemented but not cached
- All caches use **`if: always()`** for save-on-failure (BLD-05/06/07 spec)

### CUDA architectures
- Target: **`80-real;86-real;89-real;90-real;90-virtual`** (Ampere, Ada Lovelace, Hopper + PTX forward-compat)
- Drops Volta (70) and Turing (75) — covers current consumer/datacenter GPUs under the 400MB limit
- 90-virtual provides JIT fallback for future GPUs (Blackwell+)
- v2 note: architecture list could become a workflow dispatch input (noted in user_specs/002)
- CPU ISA: **disable AVX2/FMA/F16C** (`-DGGML_AVX2=off -DGGML_FMA=off -DGGML_F16C=off`) — matches upstream CUDA wheel pattern, broadest CPU compatibility since CUDA handles heavy compute

### Claude's Discretion
- vcvarsall invocation strategy (dedicated step vs inline in build step) — whatever avoids the INCLUDE/LIB overflow
- sccache setup details (hendrikmuhs/ccache-action@v1.2 configuration, cache size limits)
- Exact CMAKE_ARGS composition order and flag grouping
- Per-step timeout-minutes values (BLD-09)
- CMAKE_BUILD_PARALLEL_LEVEL value for the runner (BLD-08)
- Wheel-tag regex pattern for the assertion step (BLD-10)
- Wheel size assertion mechanism (BLD-11)
- Artifact upload configuration (BLD-13)
- Whether to use `SKBUILD_CMAKE_ARGS` instead of `CMAKE_ARGS` if scikit-build-core requires it
- If `SETUPTOOLS_SCM_PRETEND_VERSION` doesn't work with scikit-build-core, use whatever override mechanism the research finds

</decisions>

<specifics>
## Specific Ideas

- "Save caches on failure" is critical — most dev iterations ARE failing builds. The fail-fix-retry loop is the primary workflow, not the happy path.
- The `SETUPTOOLS_SCM_PRETEND_VERSION` approach may need research validation — scikit-build-core may use its own version override mechanism rather than setuptools_scm's. The key requirement is: no tracked file modifications, version override via env var.
- Upstream's `build-wheels-cuda.yaml` line 159 shows their CMAKE_ARGS pattern including the banned flag — our version replaces that with sccache flags and /Z7 enforcement instead.
- BLD-04 (`/Z7` + `CMP0141=NEW`) is load-bearing for sccache: without embedded debug info, sccache can't hash TU output deterministically and cache hit rate drops to near zero.
- Phase 1 STATE.md notes: "mamba steps use `shell: pwsh`" and "step ordering: probe-msvc FIRST for instant fail" — both carry forward into the build step.
- Phase 1 STATE.md notes: use `$env:CONDA_PREFIX\Library` for CUDA paths (not `$env:CUDA_PATH`).

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- **probe-msvc step** (already in workflow): outputs install_path, selected_full_version, selected_toolset, msc_ver_cap — build step reuses these for vcvarsall location and toolset selection
- **assert-submodule step** (already in workflow): outputs llama_sha — build step reuses for wheel version embedding
- **Upstream build-wheels-cuda.yaml** (reference): lines 113-169 show the CMAKE_ARGS construction pattern, mamba install, and `python -m build --wheel` invocation. Our version differs in: no banned flag, sccache flags added, /Z7 enforcement, Ninja generator
- **pyproject.toml**: scikit-build-core configured with `wheel.packages = ["llama_cpp"]`, `wheel.py-api = "py3"`, version from `llama_cpp/__init__.py` via regex provider
- **CMakeLists.txt**: installs llama, ggml, ggml-base, ggml-cuda targets to `llama_cpp/lib/`. Windows-specific: `WINDOWS_EXPORT_ALL_SYMBOLS ON`, `TARGET_RUNTIME_DLLS` copied for CUDA DLL bundling (lines 102-138)

### Established Patterns
- All Windows steps use `shell: pwsh` (Phase 1 convention, matches upstream)
- `set -euo pipefail` prelude for all bash blocks (Phase 1 convention)
- `Set-StrictMode -Version Latest; $ErrorActionPreference = 'Stop'` prelude for all pwsh blocks
- External actions pinned with `@v4`/`@v5` semantic tags
- Hard-fail with remediation hints on any assertion failure (Phase 1 pattern)

### Integration Points
- Build steps are in the `build` job in the workflow YAML (renamed from `preflight` during Phase 2 execution — the job now contains both toolchain preflight assertions AND build steps)
- probe-msvc outputs consumed by build step for vcvarsall path
- assert-submodule outputs consumed for version SHA embedding
- Build produces wheel artifact that Phase 3 (smoke-test) will download
- Cache keys may reference the same hashFiles patterns used by Phase 1's forensics

</code_context>

<deferred>
## Deferred Ideas

- **CUDA architectures as dispatch input** — noted in .planning/user_specs/002_V2_AUTO_TOOLCHAIN_SELECTION.md for v2. Would let users trade wheel size for GPU coverage.
- **delvewheel repair** — vendor non-CUDA transitive DLLs (VC++ runtime) into the wheel (QA-04, v2 scope)
- **Multi-Python matrix** — 3.10/3.11/3.12 builds (MX-01, v2 scope)
- **SBOM generation** — CycloneDX alongside wheel (QA-02, v2 scope)

</deferred>

---

*Phase: 02-build-cache*
*Context gathered: 2026-04-16*
