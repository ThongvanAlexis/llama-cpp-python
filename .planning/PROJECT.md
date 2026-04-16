# Windows CUDA Wheels CI for llama-cpp-python (fork)

## What This Is

A GitHub Actions workflow, living in this fork of `abetlen/llama-cpp-python`, that builds and publishes Windows x64 CUDA wheels of `llama-cpp-python` on demand. The upstream maintainer stopped publishing Windows CUDA wheels in July 2025 (commit `98fda8c`, "Temporarily" — still off), leaving Windows users stuck at 0.3.4 and unable to run modern models (Qwen3, Gemma 3, etc.) with CUDA acceleration. This fork reactivates that path for our own use, published as a pip-compatible index on GitHub Pages.

## Core Value

**Produce a Windows x64 CUDA wheel that actually works at runtime** (loads a model, runs inference without segfault) and is installable via `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu126`.

The runtime-works part is non-negotiable: upstream's historical failure mode is wheels that install fine but segfault on first `Llama(...)` call when `-allow-unsupported-compiler` was used to paper over VS/nvcc incompatibility. Every publish must be gated on a passing smoke test.

## Requirements

### Validated

<!-- Inferred from existing codebase — the llama-cpp-python library itself already works; we're adding CI. -->

- ✓ Python bindings to llama.cpp C++ library (ctypes + vendored llama.cpp submodule) — upstream
- ✓ CMake + scikit-build-core build system producing platform wheels — upstream
- ✓ Existing cross-platform CI workflow `.github/workflows/build-wheels-cuda.yaml` with Windows steps already written but commented out — upstream
- ✓ Existing test suite under `tests/` (pytest) — upstream

### Active

<!-- New scope for this fork. Each becomes a phase input. -->

- [ ] New workflow file `.github/workflows/build-wheels-cuda-windows.yaml` dedicated to Windows (avoid upstream merge conflicts)
- [ ] Matrix reduced to Windows-only; `python_version` and `cuda_version` exposed as `workflow_dispatch` inputs (defaults: 3.11, 12.6.3 — bumped from 12.4.1 after OQ1 resolution, see Key Decisions)
- [ ] VS 2022 paths and `vs-version` range adapted from upstream's VS 2019 assumptions; VS version pinned to an nvcc-12.4-compatible range
- [ ] sccache caching wired into CMake (C / CXX / CUDA compiler launchers)
- [ ] CUDA installer zip cached (`actions/cache@v4`, keyed on CUDA version)
- [ ] mamba/conda package cache for `cuda-toolkit` install
- [ ] Caches saved on build failure (dev-loop speed: fast recovery when builds break)
- [ ] Test fixture `llm_for_ci_test/tiny-llama.gguf` (27KB) committed to repo
- [ ] Smoke-test step: install built wheel, `from llama_cpp import Llama`, load tiny-llama.gguf, generate 2 tokens, assert no crash — runs on Windows runner before publish
- [ ] Publish path: GitHub Pages `gh-pages` branch, pip-compatible index at `whl/cu<major><minor>/llama-cpp-python/index.html` pointing at the wheel asset
- [ ] `workflow_dispatch`-only trigger (no automatic tag/release triggers in v1)
- [ ] README or docs note on how to install via the published index

### Out of Scope

- **Linux wheels** — upstream still publishes these; no reason to duplicate
- **macOS / Metal wheels** — upstream still publishes these
- **Automatic trigger on tag/release push** — deferred until the manual pipeline is stable (v2)
- **Full Python × CUDA matrix in one run** — dispatch inputs let a user pick one combo per run; sweeping the full matrix is out of scope for v1 (would blow the minutes budget while we're still fighting VS/nvcc)
- **Upstream PR to reactivate Windows in abetlen/llama-cpp-python** — upstream is in maintenance pause (see #2136, Quansight funding offer with no response); not worth the effort until we have a track record
- **`-allow-unsupported-compiler` fallback** — banned. Produces wheels that segfault at runtime; better to fail the build loudly and fix the VS pin
- **CPU-only wheels** — CUDA only; CPU Windows wheels exist via `pip install llama-cpp-python` already
- **Python 3.13 / free-threaded builds** — not our immediate need; upstream itself hasn't solved it (#2136)

## Context

**Why this exists (user spec, `.planning/user_specs/001_WINDOWS_CUDA_WHEELS_CI_PLAN.md`):**
- Upstream abandoned Windows CUDA wheels in July 2025 after a runner-image churn week; no explicit technical reason, but the build has been flaky since early 2024 over VS/nvcc version matching.
- The community fallback index `jllllll.github.io/llama-cpp-python-cuBLAS-wheels` has been frozen at 0.2.26 since Jan 2024.
- Downstream consumers (APHP project using this fork) need modern llama.cpp features (Qwen3, Gemma 3, flash attention) on Windows + CUDA.

**Technical environment:**
- Fork at `C:\claude_checkouts\llama-cpp-python` (Windows 10 dev host, git already initialized, on `main` branch).
- Upstream's existing `build-wheels-cuda.yaml` contains most of the Windows build steps (MSBuild install, CUDA toolkit via Jimver + mamba, VS MSBuildExtensions copy) — `~80%` of the work is already written, just gated behind a commented `# 'windows-2022'` in the OS matrix.
- Reference caching pattern from `C:\claude_checkouts\keepassxc\.github\workflows\build.yml` (sccache setup + vcpkg pattern).

**Known failure modes upstream (read sections 8 & 9 of the user spec):**
1. Runner VS image rotation breaks nvcc's strict compiler-version check (`unsupported Microsoft Visual Studio version!`) — issues #1543, #1551, #1838, #1894.
2. `-allow-unsupported-compiler` gets past compile but yields runtime-segfault wheels. Do not use.
3. Upstream workflow points at VS 2019 Enterprise paths; `windows-2022` runner has VS 2022. Paths and `vs-version` range both need updating.
4. CUDA 12.5+ was removed from upstream matrix for the same VS-compatibility reason (commit `4f17ae5`).

## Constraints

- **Tech stack**: GitHub Actions YAML only. No custom runners. No shell scripts outside the workflow (keep debugging surface small).
- **Platform**: Windows x64 runner (`windows-2022`, possibly `windows-2025` if VS 17.x / nvcc 12.4 alignment forces it).
- **Python target (v1)**: 3.11 default, but 3.10/3.12/3.13 selectable via dispatch input.
- **CUDA target (v1)**: 12.6.3 default (compatible with driver ≥ 560.x on Windows per NVIDIA compatibility matrix). Dispatch input allows other 12.x. Bumped from 12.4.1 on 2026-04-15 after OQ1 resolved: VC.14.39.17.9 component no longer offered on windows-2022 image (actions/runner-images#9701); MSVC 14.40 is the lowest-retired-safe toolset and requires nvcc `_MSC_VER` cap ≥ 1940, which starts at CUDA 12.5.
- **Build budget**: ≤ 6h per job (GitHub free-tier runner timeout). Expected: 30–45 min cold, 5–10 min warm after caches.
- **Wheel size**: can exceed 200 MB; fits GitHub Release (2 GB) and gh-pages limits.
- **No reliance on upstream merge**: workflow lives in a separate file so rebases from upstream don't stomp it.
- **Smoke test gates publish**: a wheel that compiles but segfaults must never land on `gh-pages`.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| New file `build-wheels-cuda-windows.yaml`, not edit existing | Avoid upstream merge conflicts; upstream still owns `build-wheels-cuda.yaml` | — Pending |
| Publish via GitHub Pages pip-compatible index on `gh-pages` | Mirrors abetlen's original UX (`--extra-index-url`); makes `pip install llama-cpp-python` Just Work | — Pending |
| Workflow trigger: `workflow_dispatch` only in v1 | Minimize noise while stabilizing the VS/nvcc fight; tag/release triggers deferred to v2 | — Pending |
| Commit `llm_for_ci_test/tiny-llama.gguf` (27 KB) into the repo | Simplest smoke-test plumbing; size is negligible; CI just checks it out | — Pending |
| Dispatch inputs for `python_version` + `cuda_version` (defaults 3.11 / 12.6.3; msvc_toolset auto) | Lets us rebuild for different combos without editing YAML; keeps v1 scope small. msvc_toolset changed from '14.40' to 'auto' on 2026-04-16 (auto-select design). | Complete (2026-04-16) |
| Bump default from CUDA 12.4.1 → 12.6.3 and MSVC 14.39 → 14.40 (OQ1 resolution) | `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` removed from windows-2022 image (actions/runner-images#9701). 14.40 requires nvcc `_MSC_VER` cap ≥ 1940, available starting CUDA 12.5; 12.6.3 is the most recent 12.6 patch confirmed available on the `nvidia/label/cuda-12.6.3` conda channel. | Complete (2026-04-15) — preflight diagnostic hard-failed at probe step with remediation hint; acceptable behavior. |
| Ban `-allow-unsupported-compiler` | Historical failure mode: compiles but segfaults at runtime | — Pending |
| Smoke test is a publish gate, not an advisory step | The *whole point* of the project is wheels that work at runtime, not wheels that install | — Pending |
| Three extra caches: sccache, CUDA installer zip, mamba pkgs | Keepassxc-style cache stack; target cold 30–45 min → warm 5–10 min | — Pending |
| Save caches on build failure | Most dev iterations will be failing builds; don't re-pay download cost every attempt | — Pending |
| Enumerate runner MSVC toolsets from disk, not install via vs_installer | Both 14.39 and 14.40 VC components retired from windows-2022 channel manifest (actions/runner-images#9701). Enumeration of `VC\Tools\MSVC\*` is instant, authoritative, and self-documenting. If pin not found, fail with list of available pins. | Complete (2026-04-16) |
| Auto-select MSVC from CUDA compat matrix | msvc_toolset defaults to 'auto'; probe step uses CUDA<->MSVC compatibility matrix (host_config.h _MSC_VER caps) to pick newest compatible toolset from whatever is installed. Corrected cap model: CUDA 12.4-12.9 all cap at _MSC_VER < 1950 (not per-minor as assumed). Resilient to runner image rotation. | Complete (2026-04-16) |

---
*Last updated: 2026-04-16 — MSVC auto-select from CUDA compat matrix; msvc_toolset defaults to 'auto'; resilient to runner image rotation*
