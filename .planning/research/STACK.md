# Stack Research — Windows CUDA Wheel CI for llama-cpp-python

**Domain:** GitHub Actions CI for building & publishing Windows x64 CUDA Python wheels (llama-cpp-python fork)
**Researched:** 2026-04-15
**Confidence:** HIGH on action versions and VS/CUDA compatibility; MEDIUM on `windows-2025-vs2026` beta image behaviour (not exercised at publish time)

---

## Executive Summary

The upstream workflow at `.github/workflows/build-wheels-cuda.yaml` already contains most of what we need, but it targets the **`windows-2022` / VS 2019** combination that no longer exists — **the current `windows-2022` runner image ships VS 2022 17.14.37111.16 with MSVC toolset 14.50 (`_MSC_VER = 1950`)** (`Windows2022-Readme.md`, April 2026). CUDA 12.4.1's `crt/host_config.h` explicitly rejects any `_MSC_VER >= 1950`, so *simply uncommenting `'windows-2022'` will always produce the exact "unsupported Microsoft Visual Studio version!" error that upstream has been chasing since 2024*. The fix is not a runner choice, it's a **toolset pin**: install MSVC v14.39 as a side-by-side toolset component in VS 2022, then activate it via `ilammy/msvc-dev-cmd` with `toolset: 14.39`. That's the pattern ExLlamaV2 and PyTorch's Windows CUDA wheel builds use, and it's the only reliable way to satisfy nvcc 12.4.1 without `-allow-unsupported-compiler` (which we've banned because it produces wheels that segfault at runtime).

The other stack decisions are much more boring. `Jimver/cuda-toolkit` + `conda-incubator/setup-miniconda` (mamba) is the standard two-step Windows CUDA install used by upstream and by flash-attention, qiskit-aer and similar "wheel factory" projects. `hendrikmuhs/ccache-action@v1.2` covers sccache on Windows (proven in keepassxc's workflow). `actions/cache@v4` is the correct pinned version for caching the ~3 GB CUDA installer zip and mamba's `~/conda_pkgs_dir`. For publish, `peaceiris/actions-gh-pages@v4` with `keep_files: true` and `destination_dir: whl/cu124/llama-cpp-python/` is the right choice — `actions/deploy-pages` is built for "single atomic site replacement" and cannot accumulate wheels across runs on a `gh-pages` branch.

---

## Recommended Stack

### Core Actions

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **actions/checkout** | `@v4` | Clone repo with `submodules: recursive` | v4 is Node 20, current stable; v5 exists but brings no relevant behaviour change for submodule+LFS flows. Upstream already uses v4. |
| **actions/setup-python** | `@v5` | Install Python 3.11 interpreter + pip cache | v5 is the long-lived stable line; v6 (Node 24) released Jan 2025 requires runner `>=2.327.1` and adds no features we need. v5's `cache: 'pip'` already works. |
| **conda-incubator/setup-miniconda** | `@v3` | Install mamba for the `cuda-toolkit` conda package | Used by upstream workflow today (`@v3.1.0`). `miniforge-version: latest` pulls the latest mambaforge at job time. Lets nvcc + cudart live in a self-contained `CONDA_PREFIX` instead of polluting the system. |
| **Jimver/cuda-toolkit** | `@v0.2.24` | Download the CUDA 12.4.1 installer zip and extract the VS MSBuildExtensions | Pinned to `v0.2.24` (June 2024) specifically — later `v0.2.25+` moved to node24 and shuffled the `windows-links.ts` format that upstream's inline `Invoke-RestMethod` parser depends on. We only use this for the link table and the MSBuildExtensions; we don't invoke the action's main installer (issue #382 documents its 12.5.x hang on `windows-2022`). |
| **ilammy/msvc-dev-cmd** | `@v1` | Activate a *specific* MSVC toolset (14.39 for CUDA 12.4) in the job's shell | **This is the new, critical piece.** `microsoft/setup-msbuild` only discovers already-installed VS versions — its `vs-version` input cannot downgrade a 17.14 runner to 17.9. `ilammy/msvc-dev-cmd` with `toolset: 14.39` runs `vcvarsall.bat -vcvars_ver=14.39` which is what's actually needed. |
| **microsoft/setup-msbuild** | `@v2` | Make `msbuild.exe` available on PATH for the VS integration copy step | v2 is Node 20, stable. We keep it for the MSBuild-discovery step; the actual compile uses the environment set up by `ilammy/msvc-dev-cmd`. v3 exists but just bumps to Node 24 — no behaviour change. |
| **hendrikmuhs/ccache-action** | `@v1.2` | Compiler cache (sccache variant) for C/C++/CUDA | Current major: `v1.2.22` (March 2026). Supports `variant: sccache` on Windows with explicit ARM64 and macOS binary shipping since v1.2.21. Proven in keepassxc's build.yml. Float on `v1.2` — point releases are safe. |
| **actions/cache** | `@v4` | Cache the CUDA installer zip (~3 GB) and the mamba pkg dir | v4 uses the new cache service API. v5 (`v5.0.5`, April 2026) is also backward compatible and requires runner >=2.327.1; v4 is the conservative pin and matches what upstream already uses. **Do not use v3** — v3's cache backend was sunset Feb 2025. |
| **peaceiris/actions-gh-pages** | `@v4.0.0` | Publish the built wheel + `index.html` to `gh-pages` branch with per-CUDA-version subdirectories | Only option that supports `keep_files: true` + `destination_dir: whl/cu124/llama-cpp-python/` — the exact UX needed for a pip-compatible index that accumulates wheels across many runs. v4 (April 2024) updated to Node 20. Works with a write-scoped `GITHUB_TOKEN`; no PAT needed after the first manual branch setup. |
| **softprops/action-gh-release** | `@v2` | Upload wheels as release assets (secondary channel) | Already in the upstream workflow. Fine as-is for tag-triggered releases, but irrelevant for v1 (v1 is `workflow_dispatch` only). Keep for future v2 tag trigger. |

### Runner

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **runs-on: `windows-2022`** | image version `>= 2026041x` | Primary build host | Ships VS 2022 Enterprise 17.14.37111.16, MSVC runtime 14.50, no preinstalled CUDA toolkit. We **install** the CUDA toolkit ourselves via Jimver + mamba, and we **downgrade** the active MSVC toolset at compile time to 14.39. Do not use `windows-latest` — GitHub has announced it will point to `windows-2025` over 2026, which would silently change the VS baseline under our feet. |
| **Alternative: `windows-2025`** | for future | Newer OS; still VS 2022 17.14 baseline | Currently equivalent to `windows-2022` for our purposes; switch if/when `windows-2022` is retired. |
| **Avoid: `windows-2025-vs2026`** | beta | VS 2026 based | VS 2026 / MSVC v144 (`_MSC_VER >= 1950`) will fail every CUDA version we care about. Useful only for CUDA 13.x experimentation later. |

### Build / Toolchain Stack (driven from the workflow, not installed by actions)

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **Visual Studio 2022 Enterprise** | 17.14.x (preinstalled) | Host for MSBuild, installer, workloads | Preinstalled on the runner; we don't pick it, it picks us. The only knob we touch is the **active toolset**. |
| **MSVC toolset** | `v14.39` (`_MSC_VER = 1939`) | Host C/C++ compiler nvcc actually invokes | Matches CUDA 12.4.1's `host_config.h` support band (`_MSC_VER` in `[1910, 1950)`). `14.39.x` corresponds to VS 2022 17.9's toolset — confirmed in Microsoft's "MSVC Toolset Minor Version Number 14.40" devblog. Installed via **`Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64`** workload ID through `vs_installer.exe modify` (or preinstalled on `windows-2022` — check image readme in roadmap phase). |
| **CUDA Toolkit** | `12.4.1` (default) | nvcc + cudart + cuBLAS + cuRAND | Targeted by the project spec. Compatible with user drivers `>= 525.x` (forward compat) so covers RTX 20/30/40 and any 2026 Blackwell card via driver layer. |
| **CMake** | `>= 3.21` (preinstalled on runner) | Generate MSBuild projects from `CMakeLists.txt` | Preinstalled version on `windows-2022` is ≥ 3.27; no setup action needed. llama.cpp requires 3.21+. |
| **scikit-build-core** | `>= 0.10.0` (from `pyproject.toml`) | PEP 517 build backend mediating between `python -m build` and CMake | Already the project's backend. Accepts `CMAKE_ARGS` env var and passes through `-D...` flags verbatim to CMake. |
| **python -m build** | `>= 1.2` | PEP 517 frontend that produces the `.whl` | **Preferred over cibuildwheel for this project.** See "What NOT to Use" section — cibuildwheel has no first-class Windows CUDA story. |
| **sccache** | `>= 0.8` (auto-installed by ccache-action) | Object-file cache shared across `cl.exe` and `nvcc` invocations | Works with `CMAKE_{C,CXX,CUDA}_COMPILER_LAUNCHER=sccache`. The CUDA launcher support landed in CMake 3.27 so our generator must be Ninja or we must use MSBuild ≥ 17.5 for the launcher indirection to work on the CUDA rule. |

---

## Installation (workflow snippets)

### CUDA / MSVC pinning (the load-bearing steps)

```yaml
# 1) The VS installer already has VS 2022 17.14 + MSVC 14.50 on disk.
#    We explicitly install the older 14.39 toolset (one-time per job, ~2 min).
- name: Install MSVC 14.39 toolset (CUDA 12.4 compat)
  if: runner.os == 'Windows'
  run: |
    Set-Location "C:\Program Files (x86)\Microsoft Visual Studio\Installer"
    $vsPath = & .\vswhere.exe -latest -property installationPath
    Start-Process -Wait -FilePath "$vsPath\..\..\..\..\Installer\setup.exe" -ArgumentList `
      "modify","--installPath","`"$vsPath`"","--quiet","--norestart", `
      "--add","Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64"

# 2) Activate the 14.39 toolset in the shell (affects later cl.exe + nvcc invocations)
- name: Enable VS build environment with MSVC 14.39
  if: runner.os == 'Windows'
  uses: ilammy/msvc-dev-cmd@v1
  with:
    toolset: 14.39
    arch: x64
```

### Caching trio

```yaml
# A. Compiler cache
- name: Set up sccache
  id: sccache
  uses: hendrikmuhs/ccache-action@v1.2
  with:
    variant: sccache
    key: windows-llama-cpp-cuda-${{ matrix.cuda }}-py${{ matrix.pyver }}
    append-timestamp: false

# B. CUDA installer zip (3 GB, changes only per CUDA version)
- name: Cache CUDA installer
  id: cuda-zip-cache
  uses: actions/cache@v4
  with:
    path: cudainstaller.zip
    key: cuda-installer-${{ matrix.cuda }}-windows-v1

# C. mamba package cache
- name: Cache mamba packages
  uses: actions/cache@v4
  with:
    path: |
      ~/conda_pkgs_dir
      C:\Miniconda3\pkgs
    key: mamba-cuda-${{ matrix.cuda }}-win-v1
    restore-keys: |
      mamba-cuda-${{ matrix.cuda }}-win-
      mamba-cuda-
```

**Critical:** save caches *on failure* too (speed up the fail-fix-retry dev loop):

```yaml
- name: Save CUDA zip cache on failure
  if: "failure() && steps.cuda-zip-cache.outputs.cache-hit != 'true'"
  uses: actions/cache/save@v4
  with:
    path: cudainstaller.zip
    key: cuda-installer-${{ matrix.cuda }}-windows-v1
```

### CMake args (scikit-build-core path)

```powershell
$env:CMAKE_ARGS = @(
  "-DGGML_CUDA=on",
  "-DGGML_CUDA_FORCE_MMQ=ON",
  "-DCMAKE_CUDA_ARCHITECTURES=70-real;75-real;80-real;86-real;89-real;90-real;90-virtual",
  "-DCMAKE_C_COMPILER_LAUNCHER=sccache",
  "-DCMAKE_CXX_COMPILER_LAUNCHER=sccache",
  "-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache"
) -join " "
python -m build --wheel
```

The existing upstream workflow already passes these via `CMAKE_ARGS`; scikit-build-core forwards that env var to CMake unchanged. **Do not** add `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` — that's the historical foot-gun (issue #1543) that produces wheels which segfault at runtime.

### gh-pages publish

```yaml
- name: Build pip index page
  run: |
    $wheel = Get-ChildItem dist\*.whl | Select-Object -First 1
    New-Item -ItemType Directory -Force -Path "pages\whl\cu124\llama-cpp-python"
    Copy-Item $wheel.FullName "pages\whl\cu124\llama-cpp-python\"
    # Minimal PEP 503 index: just <a href="wheel-name.whl">wheel-name.whl</a>
    # (index merged with existing files on gh-pages thanks to keep_files=true)
    @"
    <!DOCTYPE html><html><body>
    <a href="$($wheel.Name)">$($wheel.Name)</a>
    </body></html>
    "@ | Set-Content "pages\whl\cu124\llama-cpp-python\index.html"

- name: Deploy to gh-pages
  uses: peaceiris/actions-gh-pages@v4
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./pages
    keep_files: true   # do NOT wipe other cuXYZ directories
    commit_message: "publish ${{ env.WHEEL_NAME }}"
```

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| **`python -m build`** (scikit-build-core direct) | `cibuildwheel` | If we ever needed the manylinux/musllinux auditwheel flow, or Python version matrixing via `CIBW_BUILD`. For Windows CUDA, cibuildwheel buys nothing: there is no Windows equivalent of `auditwheel repair`, no GPU smoke-test story, and cibuildwheel spawns an extra Docker-ish layer we don't want on Windows. Confirmed by the ExLlamaV2 / qiskit-aer / flash-attention-prebuild-wheels projects all using `python -m build` directly for Windows CUDA. |
| **`peaceiris/actions-gh-pages@v4`** | `actions/deploy-pages` | If we were publishing a full static site that's regenerated atomically each run (docs site, single-page SPA). `actions/deploy-pages` requires a single artifact upload per deploy; it has no "add this file, keep the others" semantics, which is the entire point of a pip-compatible index that spans multiple `cuXYZ` subdirs. |
| **`peaceiris/actions-gh-pages@v4`** | `JamesIves/github-pages-deploy-action@v4` | JamesIves is functionally equivalent and slightly more popular, but peaceiris has the cleaner `keep_files` + `destination_dir` combination and is what most scientific wheel repos (llama-cpp-python upstream pre-2025, jllllll's old index) actually used. Either is defensible. |
| **`Jimver/cuda-toolkit@v0.2.24` (partial use)** | `Jimver/cuda-toolkit@v0.2.35` (full use, installs toolkit) | The later action's full installer path is what issue #382 says hangs on `windows-2022`. Our workaround — download zip via Jimver's link table, install actual nvcc via mamba — sidesteps that. |
| **`conda-incubator/setup-miniconda` + `mamba install cuda-toolkit`** | Direct download of NVIDIA Windows network installer | The NVIDIA `.exe` installer is interactive-first, its silent-install flags (`-s nvcc_12.4`) change between CUDA minor versions, and it writes VS integration into whichever VS it finds (including the wrong toolset). The conda path puts nvcc into `$CONDA_PREFIX` cleanly and matches how upstream's Linux path works. |
| **`ilammy/msvc-dev-cmd@v1` with `toolset: 14.39`** | `microsoft/setup-msbuild@v2` with `vs-version: '[17.9,17.10)'` | `microsoft/setup-msbuild` only selects among installed VS versions (confirmed in action's README — "scoped to what software actually exists on the runner image"). Since the runner has only 17.14, setting `vs-version: '[17.9,17.10)'` fails. `ilammy/msvc-dev-cmd` runs `vcvarsall.bat -vcvars_ver=14.39`, which *does* work once the 14.39 toolset component has been installed via `vs_installer.exe modify`. |
| **`hendrikmuhs/ccache-action@v1.2` (sccache variant)** | `Mozilla/sccache-action` or manual `choco install sccache` | `hendrikmuhs/ccache-action` handles the GitHub cache wiring, reports hit/miss stats, and supports both `ccache` and `sccache` under one variable. Used by keepassxc's production build.yml. The other options require reimplementing the cache save/restore logic. |
| **`actions/cache@v4`** | `actions/cache@v5` | v5 is newer (April 2026) and backward-compatible, but requires a newer runner version. v4 is the safer pin, zero-benefit difference for our ~3 GB blob. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **`CMAKE_CUDA_FLAGS=--allow-unsupported-compiler`** | *The* historical failure mode: compiles fine, produces a `.whl` that installs fine, then **segfaults at runtime on first `Llama(...)`**. Confirmed in llama-cpp-python issue #1543 and NVIDIA forum thread "MSC_VER is 1940... llama.cpp". Upstream's own 2025-07 commit (`98fda8c`) is the result of this exact trap. | Pin MSVC toolset to 14.39 via `ilammy/msvc-dev-cmd`. If the pin breaks because a future CUDA needs a newer MSVC, raise the pin — don't bypass the check. |
| **`windows-latest` runner label** | GitHub is in the middle of repointing `windows-latest` from `windows-2022` to `windows-2025` (issue `actions/runner-images#12677`). Our toolset pin relies on knowing exactly which VS side-by-side components are installable on the image. A silent label-flip would break the build without any code change. | Pin `runs-on: windows-2022`. |
| **`actions/setup-python@v4` or older** | Reached end of GitHub support; pip cache behaviour changed in v5. Also uses Node 16, which is deprecated for GitHub Actions. | `actions/setup-python@v5`. |
| **`actions/cache@v3`** | Old cache backend sunset on Feb 1, 2025. Workflows using it either fail or silently skip caching. | `actions/cache@v4`. |
| **`microsoft/setup-msbuild@v2` with `vs-version: '[17.9,17.10)'`** | **Does not install 17.9.** The input selects among already-installed VS versions. On a `windows-2022` runner with only 17.14, this step will either no-op or fail the step. The upstream workflow's `vs-version: '[16.11,16.12)'` (VS 2019) was a leftover from when `windows-2019` runners still existed — it's already dead code. | Install 14.39 component via `vs_installer.exe modify`, then activate with `ilammy/msvc-dev-cmd@v1` `toolset: 14.39`. |
| **`Jimver/cuda-toolkit` full installer path on CUDA 12.5+** | Issue #382 ("Windows 2022 Runner - Cuda 12.5.x installer hangs"): installer hangs for >= 90 min, causes job timeout. Root cause is an NVIDIA network installer change that the action hasn't worked around. | Use the action only to read its `windows-links.ts` for the local-installer URL, then do an explicit `Invoke-RestMethod` download (what upstream already does) + mamba for the actual nvcc install. For CUDA 12.4.1 specifically the full action is also fine, but the split pattern is the one that will keep working across 12.5/12.6/12.8. |
| **`cibuildwheel` for Windows CUDA** | No Windows equivalent of auditwheel; cibuildwheel's Windows path is effectively a wrapper around `python -m build` plus Python version matrix handling, and CUDA toolkit install is up to you anyway. Real Windows CUDA wheel projects (ExLlamaV2, flash-attention-prebuild-wheels, qiskit-aer Windows) use `python -m build` directly. | `python -m build --wheel` with scikit-build-core as the backend. |
| **`actions/deploy-pages`** | Designed for atomic full-site replacement via `actions/upload-pages-artifact`. Cannot accumulate files across runs — a second `workflow_dispatch` with a different CUDA version would wipe the first wheel's directory. | `peaceiris/actions-gh-pages@v4` with `keep_files: true`. |
| **`-DCMAKE_CUDA_ARCHITECTURES=native`** | GitHub Actions Windows runners have **no GPU**, so `native` detection produces an empty/degenerate architecture list and the wheel silently excludes any real-device cubins. | Explicit list: `70-real;75-real;80-real;86-real;89-real;90-real;90-virtual` (same as upstream, covers Volta through Hopper + one forward-compat PTX). |

---

## Stack Patterns by Variant

**If CUDA target is `12.4.1` (default, v1 scope):**
- MSVC toolset pin: **14.39** (`_MSC_VER = 1939`)
- Required VS component: `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64`
- Driver compatibility: `>= 525.60.13` on Windows
- Works on `windows-2022`: yes (with toolset install step)

**If CUDA target is bumped to `12.6.x` later:**
- MSVC toolset pin: **14.40** or **14.41** (both accepted by CUDA 12.5+ which raised the ceiling to `_MSC_VER < 1950`)
- Required VS component: `Microsoft.VisualStudio.Component.VC.14.40.17.10.x86.x64`
- Jimver full-installer path becomes risky (issue #382) — keep the zip-download-only pattern

**If CUDA target is `12.8+`:**
- Still `_MSC_VER < 1950` ceiling in current host_config.h (CUDA 12.8 does not raise it)
- Hand-inspect `crt/host_config.h` in the downloaded CUDA installer before committing to a toolset pin

**If we ever need Python 3.13:**
- Upstream project itself hasn't solved 3.13 yet (issue #2136); blocked by scikit-build-core + ctypes-extensions work, not by our CI
- Out of scope for this phase

---

## Version Compatibility Matrix

### CUDA toolkit ↔ MSVC toolset (`_MSC_VER` range from `crt/host_config.h`)

| CUDA | Accepts `_MSC_VER` | MSVC toolset versions | VS 2022 versions | Confidence |
|------|--------------------|------------------------|-------------------|------------|
| 12.2.x | `[1910, 1940)` | 14.20 – 14.39 | 17.0 – 17.9 | HIGH (NVIDIA forum thread) |
| **12.4.0 / 12.4.1** | `[1910, 1950)` | 14.20 – **14.4x** | **17.0 – 17.9 confirmed**; 17.10 reported working but untested by us | HIGH (Microsoft STL issue #4941 + NVIDIA forum) |
| 12.5.0 | `[1910, 1950)` | 14.20 – 14.4x | 17.0 – 17.10 | MEDIUM (one NVIDIA-forum source) |
| 12.6.x | `[1910, 1950)` | 14.20 – 14.4x | 17.0 – 17.10 | MEDIUM |
| 12.8+ | `[1910, 1950)` | 14.20 – 14.4x | 17.0 – 17.10 | LOW (no authoritative recent source examined) |
| 13.0+ | `[1910, ???)` | — | VS 2026 support added in 13.x | OUT OF SCOPE |

**Key takeaway:** for CUDA 12.4.1 specifically, the **safe and conservatively-proven** toolset is **14.39**. 14.40 should also work but we haven't verified it for this project; keep 14.39 as the default and only bump if a future CUDA forces it.

### Action version matrix (pinned for v1 of the workflow)

| Action | Pin | Latest available | Rationale for pin |
|--------|-----|------------------|-------------------|
| actions/checkout | `v4` | v4 (current) | Stable |
| actions/setup-python | `v5` | v6.2.0 | v5 is Node 20; v6 is Node 24 and needs runner `>=2.327.1`. Zero behaviour change for us. |
| conda-incubator/setup-miniconda | `v3` | v3.1.x | Match upstream |
| Jimver/cuda-toolkit | `v0.2.24` | v0.2.35 | v0.2.25+ changes `windows-links.ts` format the upstream workflow parses inline |
| ilammy/msvc-dev-cmd | `v1` | v1 (floating) | Float on major — single purpose action, unlikely to regress |
| microsoft/setup-msbuild | `v2` | v3 | v3 is Node 24 bump only |
| hendrikmuhs/ccache-action | `v1.2` | v1.2.22 | Float on minor — point releases are safe |
| actions/cache | `v4` | v5.0.5 | v5 requires newer runner; v4 sufficient |
| peaceiris/actions-gh-pages | `v4.0.0` | v4.0.0 | Latest major |
| softprops/action-gh-release | `v2` | v2 | Match upstream |

### Runner image ↔ VS version (as of April 2026)

| Runner label | OS | VS version | MSVC default toolset | `_MSC_VER` default |
|--------------|----|-----------|-----------------------|--------------------|
| `windows-2022` | Windows Server 2022 | 17.14.37111.16 | 14.50 | 1950 |
| `windows-2025` | Windows Server 2025 | 17.14.x | 14.50 | 1950 |
| `windows-2025-vs2026` (beta) | Windows Server 2025 | 2026 (v144) | 14.50+ | 1950+ |

All three require the **14.39 side-by-side toolset install** for CUDA 12.4.1 builds. We pin `windows-2022` for stability; `windows-2025` is a drop-in alternative.

---

## Sources

### NVIDIA (HIGH confidence)

- [CUDA Installation Guide for Microsoft Windows 12.4.1](https://docs.nvidia.com/cuda/archive/12.4.1/cuda-installation-guide-microsoft-windows/index.html) — supported host compilers ("MSVC 193x with VS 2022 17.x")
- [CUDA 12.4 Release Notes](https://docs.nvidia.com/cuda/archive/12.4.0/cuda-toolkit-release-notes/) — VS 2017 deprecated; no explicit upper bound published
- [NVIDIA forum: _MSC_VER is 1940 with latest VS 2022 update (llama.cpp)](https://forums.developer.nvidia.com/t/msc-ver-is-1940-with-latest-vs-2022-upate-and-its-not-letting-me-build-llama-cpp/295748) — exact `host_config.h` check `_MSC_VER < 1910 || _MSC_VER >= 1940` for CUDA 12.2; 12.4 raised the 1940 bound
- [NVIDIA forum: CUDA 12.4 host compiler targets unsupported OS with MSVC 19.29.30154](https://forums.developer.nvidia.com/t/cuda-12-4-host-compiler-targets-unsupported-os-with-msvc-19-29-30154/311432) — documents MSVC 19.29/19.33 incompatibility with CUDA 12.4

### Microsoft (HIGH confidence)

- [MSVC Toolset Minor Version Number 14.40 in VS 2022 v17.10 (devblogs)](https://devblogs.microsoft.com/cppblog/msvc-toolset-minor-version-number-14-40-in-vs-2022-v17-10/) — toolset 14.39 → VS 17.9 (`_MSC_VER = 1939`); 14.40 → VS 17.10 (`_MSC_VER = 1940`)
- [actions/runner-images — Windows2022-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md) — current `windows-2022` image: VS Enterprise 17.14.37111.16, MSVC runtime 14.50
- [actions/runner-images #12677 (windows-latest migration)](https://github.com/actions/runner-images/issues/12677) — `windows-latest` repointing to `windows-2025`
- [actions/runner-images #13638 (windows-2025-vs2026 beta)](https://github.com/actions/runner-images/issues/13638) — VS 2026 image availability

### GitHub Marketplace actions (HIGH confidence)

- [Jimver/cuda-toolkit releases](https://github.com/Jimver/cuda-toolkit/releases) — v0.2.35 current; v0.2.24 is the last before the link-format drift
- [Jimver/cuda-toolkit issue #382 (CUDA 12.5.x installer hangs on windows-2022)](https://github.com/Jimver/cuda-toolkit/issues/382) — confirms full-installer path is broken for 12.5+ and argues the zip-only pattern
- [hendrikmuhs/ccache-action releases](https://github.com/hendrikmuhs/ccache-action/releases) — v1.2.22 current; sccache variant confirmed for Windows
- [actions/cache README](https://github.com/actions/cache) — v4/v5 backward compatible, v3 sunset Feb 2025
- [actions/setup-python releases](https://github.com/actions/setup-python/releases) — v6.2.0 current; v5 still maintained
- [microsoft/setup-msbuild README](https://github.com/microsoft/setup-msbuild) — `vs-version` input selects among installed, does not install
- [ilammy/msvc-dev-cmd README](https://github.com/ilammy/msvc-dev-cmd) — `toolset: 14.XX.YYYYY` input runs `vcvarsall.bat -vcvars_ver=...`
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) — v4.0.0 (April 2024); `keep_files` + `destination_dir` behaviour

### Community / third-party (MEDIUM confidence)

- [CUDA/MSVC compatibility (Quasar Ghent)](https://quasar.ugent.be/files/doc/cuda-msvc-compatibility.html) — informal matrix, points to NVIDIA release notes as authority
- [Microsoft/STL issue #4941 "Which VS 2022 BuildTools for CUDA 11.8 and 12.4"](https://github.com/microsoft/STL/issues/4941) — confirms VS 17.9 (MSVC 14.39) is the safe recommendation for CUDA 12.4
- [nerfstudio issue #3157 (MSVC 1950 vs CUDA)](https://github.com/nerfstudio-project/nerfstudio/issues/3157) — confirms the 1950 ceiling and workaround (install older toolset component)
- [abetlen/llama-cpp-python issue #1543 (CUDA wheels not built by CI)](https://github.com/abetlen/llama-cpp-python/issues/1543) — *the* project-history reference for why `-allow-unsupported-compiler` produces runtime-segfault wheels
- [PyTorch issue #156181 (Windows CUDA 12.9.1 wheel with sccache launcher)](https://github.com/pytorch/pytorch/issues/156181) — real-world use of `-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache` in a major project's Windows CUDA wheel CI
- [keepassxc build.yml](C:\claude_checkouts\keepassxc\.github\workflows\build.yml) — reference pattern for `hendrikmuhs/ccache-action@v1.2` + `actions/cache/save@v5` on `windows-2022`

---

*Stack research for: Windows CUDA wheel CI*
*Researched: 2026-04-15*
