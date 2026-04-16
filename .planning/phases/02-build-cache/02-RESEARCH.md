# Phase 2: Build & Cache - Research

**Researched:** 2026-04-16
**Domain:** CMake CUDA wheel build on Windows via scikit-build-core, sccache integration, GitHub Actions caching
**Confidence:** MEDIUM-HIGH

## Summary

Phase 2 adds the build-and-cache steps to the existing `build-wheels-cuda-windows.yaml` workflow, extending the green preflight from Phase 1 into a complete wheel-producing pipeline. The core challenge is orchestrating four subsystems: (1) MSVC environment activation via vcvarsall.bat for Ninja+CUDA builds, (2) scikit-build-core's wheel versioning and tagging, (3) sccache for C++/CUDA compilation caching with MSVC /Z7 embedded debug info, and (4) GitHub Actions cache save-on-failure for the fail-fix-retry loop.

The most technically sensitive areas are: the wheel tag issue (pyproject.toml sets `wheel.py-api = "py3"` which would produce `py3-none-win_amd64` but the requirement demands `cp311-cp311-win_amd64`), the version override mechanism (scikit-build-core's regex provider reads from `llama_cpp/__init__.py` and needs a local+segment injection), and the CMP0141/Z7 policy which has a documented bug where it may not actually convert `/Zi` to `/Z7` requiring a manual flag replacement workaround.

**Primary recommendation:** Use a dedicated vcvarsall activation step that captures environment into `$env:GITHUB_ENV`, build with Ninja generator via `python -m build --wheel`, override wheel.py-api via `SKBUILD_WHEEL_PY_API` env var (set to empty string for cpXX tag), inject version via `__init__.py` rewrite before build (then git checkout to restore), and use split `actions/cache/restore` + `actions/cache/save` for save-on-failure semantics.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Use **Ninja** generator (not VS 2022 MSBuild) -- faster builds, better sccache integration
- Activate MSVC via **vcvarsall.bat** called in a clean pwsh subprocess before building
- **Reuse probe-msvc step outputs** (install_path, selected_full_version) to locate vcvarsall.bat -- single source of truth
- Pass CMake arguments via **`$env:CMAKE_ARGS` environment variable** (matches upstream's pattern, visible in step logs)
- Version format: **`0.3.20+cu126.ll<sha12>`** -- PEP 440 local segment with CUDA tag and 12-char llama.cpp SHA
- Override mechanism: **`SETUPTOOLS_SCM_PRETEND_VERSION`** env var set before `python -m build` -- no file modifications needed (NOTE: research found this does NOT work with scikit-build-core regex provider; see Version Override section)
- Base version: **read from `llama_cpp/__init__.py`** at build time using the same regex scikit-build-core uses
- SHA length: **12 characters** (collision-safe for the large llama.cpp repo)
- **Tight primary keys, broad restore-keys fallback** for caches
- sccache key includes: cuda_version, msvc_toolset, hashFiles('vendor/llama.cpp/CMakeLists.txt', 'vendor/llama.cpp/src/**') -- skip .git
- **BLD-05 reinterpreted**: CUDA install is mamba-only (no Jimver zip), so BLD-05's intent is fulfilled by BLD-06 (mamba pkgs cache)
- **BLD-07 (VS integration)**: skip caching -- BuildCustomizations copy takes <5 seconds; just redo it each time
- All caches use **`if: always()`** for save-on-failure
- Target CUDA architectures: **`80-real;86-real;89-real;90-real;90-virtual`** (Ampere, Ada Lovelace, Hopper + PTX forward-compat)
- CPU ISA: **disable AVX2/FMA/F16C** (`-DGGML_AVX2=off -DGGML_FMA=off -DGGML_F16C=off`)

### Claude's Discretion
- vcvarsall invocation strategy (dedicated step vs inline in build step) -- whatever avoids the INCLUDE/LIB overflow
- sccache setup details (hendrikmuhs/ccache-action@v1.2 configuration, cache size limits)
- Exact CMAKE_ARGS composition order and flag grouping
- Per-step timeout-minutes values (BLD-09)
- CMAKE_BUILD_PARALLEL_LEVEL value for the runner (BLD-08)
- Wheel-tag regex pattern for the assertion step (BLD-10)
- Wheel size assertion mechanism (BLD-11)
- Artifact upload configuration (BLD-13)
- Whether to use `SKBUILD_CMAKE_ARGS` instead of `CMAKE_ARGS` if scikit-build-core requires it
- If `SETUPTOOLS_SCM_PRETEND_VERSION` doesn't work with scikit-build-core, use whatever override mechanism the research finds

### Deferred Ideas (OUT OF SCOPE)
- **CUDA architectures as dispatch input** -- v2 scope
- **delvewheel repair** -- QA-04, v2 scope
- **Multi-Python matrix** -- MX-01, v2 scope
- **SBOM generation** -- QA-02, v2 scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| BLD-01 | Wheel via `python -m build --wheel` using scikit-build-core | scikit-build-core regex provider, CMAKE_ARGS env var pattern |
| BLD-02 | sccache via `hendrikmuhs/ccache-action@v1.2` with `variant: sccache` | Action inputs documented; Windows uses sccache variant |
| BLD-03 | sccache wired via CMAKE_C/CXX/CUDA_COMPILER_LAUNCHER | sccache supports nvcc; CMAKE_CUDA_COMPILER_LAUNCHER works |
| BLD-04 | MSVC /Z7 via CMP0141=NEW + CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded | Known bug (sccache#2244) -- needs manual /Zi->/Z7 fallback |
| BLD-05 | CUDA installer cache (reinterpreted as mamba pkgs) | Fulfilled by BLD-06 per CONTEXT.md |
| BLD-06 | Mamba/conda cache with restore-keys and if: always() | actions/cache/restore + actions/cache/save split pattern |
| BLD-07 | VS CUDA integration -- skip caching, just redo | BuildCustomizations copy <5s per CONTEXT.md |
| BLD-08 | Ninja generator with CMAKE_BUILD_PARALLEL_LEVEL | Runner has 4 vCPUs; set to 2 (CUDA compilation is memory-heavy) |
| BLD-09 | Per-step timeout-minutes | Research provides recommended values per step type |
| BLD-10 | Wheel tag assertion: cp311-cp311-win_amd64 | CRITICAL: pyproject.toml `py-api = "py3"` must be overridden |
| BLD-11 | Wheel size < 400 MB assertion | PowerShell file size check, architecture list tuned |
| BLD-12 | Version embeds llama.cpp SHA | __init__.py rewrite before build (regex provider reads it) |
| BLD-13 | Wheel uploaded as GitHub Actions artifact | actions/upload-artifact@v4 with compression-level: 0 |
</phase_requirements>

## Standard Stack

### Core
| Library/Tool | Version | Purpose | Why Standard |
|-------------|---------|---------|--------------|
| scikit-build-core | >=0.9.2 (from pyproject.toml) | Build backend: CMake -> wheel | Already configured in pyproject.toml |
| Ninja | latest (from mamba or pip) | CMake generator | Faster than MSBuild, sccache-compatible |
| sccache | latest (via hendrikmuhs/ccache-action) | Compilation cache | Supports MSVC + nvcc + GitHub Actions cache backend |
| python-build | latest | PEP 517 frontend | `python -m build --wheel` invocation |

### Supporting
| Library/Tool | Version | Purpose | When to Use |
|-------------|---------|---------|-------------|
| hendrikmuhs/ccache-action | v1.2 | GitHub Action: installs+configures sccache | BLD-02, BLD-03 |
| actions/cache/restore | v4 | Restore cache at step start | BLD-06 mamba pkgs cache |
| actions/cache/save | v4 | Save cache with if: always() | BLD-05/06 save-on-failure |
| actions/upload-artifact | v4 | Upload wheel for downstream jobs | BLD-13 |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| hendrikmuhs/ccache-action | Mozilla-Actions/sccache-action | ccache-action is more mature, handles cache key management |
| Ninja | VS 2022 MSBuild | MSBuild is native but slower, sccache integration is worse |
| actions/cache split | actions/cache@v4 unified | Unified action cannot save on failure (post-step runs after job fails) |

**Installation (in CI):**
```bash
python -m pip install build wheel ninja
```

## Architecture Patterns

### Recommended Step Ordering in Workflow

```
# Existing Phase 1 preflight steps (already in workflow):
#   probe-msvc -> long-paths -> checkout -> setup-python -> setup-mamba
#   -> install-cuda -> assert-msvc -> assert-nvcc -> assert-cuda-single-path
#   -> assert-submodule -> assert-python -> discover-vs -> forensics/remediation

# Phase 2 new steps (appended after preflight):
  1. Copy VS BuildCustomizations (CUDA -> MSBuild integration)     [BLD-07]
  2. Restore mamba pkgs cache                                       [BLD-06]
  3. Install build dependencies (pip install build wheel ninja)     [BLD-01]
  4. Setup sccache via hendrikmuhs/ccache-action                    [BLD-02]
  5. Activate MSVC + Build wheel (single step with vcvarsall)       [BLD-01,03,04,08]
  6. Assert wheel tag                                               [BLD-10]
  7. Assert wheel size                                              [BLD-11]
  8. Upload wheel artifact                                          [BLD-13]
  9. Save mamba pkgs cache (if: always())                           [BLD-06]
 10. Show sccache stats                                             [BLD-02,03]
```

### Pattern 1: vcvarsall Environment Capture in PowerShell

**What:** Import MSVC environment from vcvarsall.bat into the PowerShell session so cmake+Ninja can find cl.exe, link.exe, and CUDA tools.

**When to use:** Every build step that needs MSVC compilation.

**CRITICAL CONSTRAINT:** Must NOT use ilammy/msvc-dev-cmd (overflow risk). Must NOT run vcvarsall in a separate step (env vars don't persist across steps). Must run within the SAME step as the build command.

**Example:**
```powershell
# Source: GitHub community discussion + Phase 1 STATE.md constraint
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$installPath = '${{ steps.probe-msvc.outputs.install_path }}'
$selectedFull = '${{ steps.probe-msvc.outputs.selected_full_version }}'
$vcvarsall = Join-Path $installPath 'VC\Auxiliary\Build\vcvarsall.bat'

# Capture environment before vcvarsall
$envBefore = @{}
Get-ChildItem env: | ForEach-Object { $envBefore[$_.Name] = $_.Value }

# Run vcvarsall in cmd subprocess, capture environment after
$envAfter = cmd /c "`"$vcvarsall`" x64 -vcvars_ver=$selectedFull >nul 2>&1 && set" |
  Where-Object { $_ -match '=' } |
  ForEach-Object {
    $parts = $_ -split '=', 2
    @{ Name = $parts[0]; Value = $parts[1] }
  }

# Apply only CHANGED/NEW variables to current PowerShell session
foreach ($entry in $envAfter) {
  $name = $entry.Name
  $value = $entry.Value
  if (-not $envBefore.ContainsKey($name) -or $envBefore[$name] -ne $value) {
    Set-Item "env:$name" $value
  }
}

# Verify cl.exe is now on PATH
$clPath = (Get-Command cl.exe -ErrorAction Stop).Source
Write-Host "vcvarsall activated: cl.exe at $clPath"
```

### Pattern 2: CMAKE_ARGS Construction

**What:** Build the full CMAKE_ARGS string with all required flags.

**Example:**
```powershell
# Source: upstream build-wheels-cuda.yaml line 159 pattern (modified)
$cudaArchs = '80-real;86-real;89-real;90-real;90-virtual'

$env:CMAKE_ARGS = @(
  '-G Ninja'
  '-DGGML_CUDA=on'
  "-DCMAKE_CUDA_ARCHITECTURES=$cudaArchs"
  '-DGGML_CUDA_FORCE_MMQ=ON'
  '-DGGML_AVX2=off'
  '-DGGML_FMA=off'
  '-DGGML_F16C=off'
  '-DCMAKE_C_COMPILER_LAUNCHER=sccache'
  '-DCMAKE_CXX_COMPILER_LAUNCHER=sccache'
  '-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache'
  '-DCMAKE_POLICY_DEFAULT_CMP0141=NEW'
  '-DCMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded'
) -join ' '
```

### Pattern 3: Version Override via __init__.py Rewrite

**What:** Inject the full PEP 440 version (with local segment) into `llama_cpp/__init__.py` before building, then restore it.

**Why not SETUPTOOLS_SCM_PRETEND_VERSION:** scikit-build-core's regex provider reads directly from the file specified in `[tool.scikit-build.metadata.version]`. It does NOT use setuptools_scm at all. The `SETUPTOOLS_SCM_PRETEND_VERSION` env var has zero effect here.

**Example:**
```powershell
# Read base version from __init__.py
$initFile = 'llama_cpp/__init__.py'
$content = Get-Content $initFile -Raw
$match = [regex]::Match($content, '__version__\s*=\s*"([^"]+)"')
if (-not $match.Success) { throw "Cannot parse version from $initFile" }
$baseVersion = $match.Groups[1].Value  # e.g. "0.3.20"

# Construct full version with local segment
$llamaSha = '${{ steps.assert-submodule.outputs.llama_sha }}'
$cudaTag = 'cu126'
$fullVersion = "$baseVersion+$cudaTag.ll$llamaSha"  # e.g. "0.3.20+cu126.ll1a2b3c4d5e6f"

# Rewrite __init__.py (will be restored after build)
$newContent = $content -replace '__version__\s*=\s*"[^"]+"', "__version__ = `"$fullVersion`""
Set-Content $initFile -Value $newContent -NoNewline

# Build
python -m build --wheel

# Restore (important: don't leave modified tracked file)
git checkout -- $initFile
```

### Pattern 4: Split Cache Restore/Save for Save-on-Failure

**What:** Use `actions/cache/restore` and `actions/cache/save` as separate steps instead of the unified `actions/cache@v4`, allowing `if: always()` on the save step.

**Example:**
```yaml
- name: Restore mamba pkgs cache
  id: cache-mamba-restore
  uses: actions/cache/restore@v4
  with:
    path: ${{ env.MAMBA_ROOT_PREFIX }}/pkgs
    key: mamba-cuda-${{ inputs.cuda_version }}-${{ runner.os }}-${{ hashFiles('.github/workflows/build-wheels-cuda-windows.yaml') }}
    restore-keys: |
      mamba-cuda-${{ inputs.cuda_version }}-${{ runner.os }}-

# ... build steps ...

- name: Save mamba pkgs cache
  if: always()
  uses: actions/cache/save@v4
  with:
    path: ${{ env.MAMBA_ROOT_PREFIX }}/pkgs
    key: ${{ steps.cache-mamba-restore.outputs.cache-primary-key }}
```

### Anti-Patterns to Avoid

- **Using ilammy/msvc-dev-cmd:** Causes vcvarsall overflow with conda's activation hook. BANNED per Phase 1 STATE.md.
- **vcvarsall in a separate step:** Environment variables don't persist across GitHub Actions steps. Must be in the same step as the build command.
- **SETUPTOOLS_SCM_PRETEND_VERSION with regex provider:** Has zero effect. scikit-build-core's regex provider reads from the file directly, not from setuptools_scm.
- **Unified actions/cache@v4 for save-on-failure:** The unified action's post-step save runs after the job, but cannot be conditioned with `if: always()`. Use split restore/save instead.
- **CUDA_PATH environment variable:** Use `$env:CONDA_PREFIX\Library` for CUDA paths (per Phase 1 STATE.md). Mamba CUDA install sets CONDA_PREFIX but not CUDA_PATH.
- **hashFiles with .git paths in submodule:** `hashFiles('vendor/llama.cpp/.git/...')` behaves differently when submodule `.git` is a file vs directory. Use source file patterns instead.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| sccache install + config | Manual download + PATH setup | `hendrikmuhs/ccache-action@v1.2` with `variant: sccache` | Handles install, cache key, GH Actions cache backend, stats |
| Cache save-on-failure | Custom post-job scripts | `actions/cache/restore@v4` + `actions/cache/save@v4` split | Built-in `if: always()` on save step; stable, well-tested |
| Wheel size check | External tool | PowerShell `(Get-Item dist/*.whl).Length` | Native, no dependencies |
| Parallel build config | Environment detection | Set `CMAKE_BUILD_PARALLEL_LEVEL=2` explicitly | Runner has 4 vCPUs but CUDA compilation is memory-heavy |
| Artifact upload | Custom upload scripts | `actions/upload-artifact@v4` | Standard, handles large files up to 5 GB |

**Key insight:** The build step itself is complex enough (vcvarsall + cmake + ninja + sccache + CUDA). Every ancillary concern (caching, artifact management, compiler caching) should use existing Actions/tools.

## Common Pitfalls

### Pitfall 1: Wheel Tag Mismatch (py3-none vs cp311-cp311)

**What goes wrong:** pyproject.toml has `wheel.py-api = "py3"` which produces `py3-none-win_amd64` wheels. BLD-10 requires `cp311-cp311-win_amd64`.
**Why it happens:** llama-cpp-python uses ctypes to load shared libraries (no CPython extension module), so upstream set py-api to "py3". Upstream builds via cibuildwheel which overrides the tag. Our workflow uses `python -m build --wheel` directly.
**How to avoid:** Set `SKBUILD_WHEEL_PY_API=""` (empty string) environment variable before build, which overrides pyproject.toml's `wheel.py-api = "py3"` and produces the default cpXX tag. Alternatively, pass `-C wheel.py-api=""` as a config-setting to `python -m build`.
**Warning signs:** Wheel filename contains `py3-none` instead of `cp311-cp311`.
**Confidence:** MEDIUM -- the `SKBUILD_WHEEL_PY_API` env var override is documented in scikit-build-core's config system (all options are settable via SKBUILD_* env vars), but I have not empirically verified the empty-string behavior. **Fallback:** If empty string doesn't work, use `-C wheel.py-api=""` config-setting, or as a last resort set `SKBUILD_WHEEL_PY_API=cp311`.

### Pitfall 2: CMP0141 /Z7 Policy May Not Take Effect

**What goes wrong:** Setting `CMAKE_POLICY_DEFAULT_CMP0141=NEW` + `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded` should convert `/Zi` to `/Z7`, but mozilla/sccache#2244 reports it sometimes doesn't work.
**Why it happens:** The policy must be set BEFORE `project()` or `enable_language()`. When passed via environment through scikit-build-core, the CMakeLists.txt may have already called `project()` before the policy takes effect.
**How to avoid:** Dual approach: (1) Set the policy variables via CMAKE_ARGS, AND (2) add manual flag replacement as a safety net in CMAKE_ARGS: `-DCMAKE_C_FLAGS_INIT="/Z7"` and `-DCMAKE_CXX_FLAGS_INIT="/Z7"`. Alternatively, use a CMake toolchain file or check sccache stats after first build to verify cache hits.
**Warning signs:** `sccache --show-stats` shows near-zero cache hit rate on rebuilds.
**Confidence:** MEDIUM -- the CMP0141 approach is the standard recommendation, but the bug is real. The fallback of CMAKE_C_FLAGS_INIT="/Z7" is well-documented and reliable.

### Pitfall 3: vcvarsall Environment Lost Across Steps

**What goes wrong:** If vcvarsall.bat is called in a dedicated step, its environment changes (INCLUDE, LIB, PATH additions) are lost in the next step.
**Why it happens:** Each GitHub Actions step runs in its own shell process. Environment variable changes in one step don't carry to the next (unless written to $env:GITHUB_ENV).
**How to avoid:** Two options: (a) Call vcvarsall and build in the SAME PowerShell step (simpler, recommended), or (b) capture all env changes and write them to $env:GITHUB_ENV (complex, fragile).
**Warning signs:** CMake cannot find cl.exe or link.exe despite vcvarsall running successfully.

### Pitfall 4: INCLUDE/LIB Environment Variable Overflow

**What goes wrong:** Multiple vcvarsall invocations (conda activation + explicit call) accumulate duplicate entries in INCLUDE/LIB, exceeding CMD's 8192-char limit.
**Why it happens:** Conda's activation hook may internally invoke vcvarsall. A second explicit vcvarsall call doubles the entries.
**How to avoid:** Call vcvarsall ONCE, AFTER conda/mamba setup, in a clean pwsh subprocess. The Pattern 1 approach above captures env from cmd subprocess without contaminating the conda-activated environment.
**Warning signs:** Build fails with obscure "cannot open include file" errors despite correct INCLUDE paths.

### Pitfall 5: Version Override Mechanism Mismatch

**What goes wrong:** Setting `SETUPTOOLS_SCM_PRETEND_VERSION` has no effect because scikit-build-core uses the regex provider, not setuptools_scm.
**Why it happens:** The pyproject.toml configures `[tool.scikit-build.metadata.version]` with `provider = "scikit_build_core.metadata.regex"` and `input = "llama_cpp/__init__.py"`. This reads the version directly from the file, ignoring setuptools_scm env vars.
**How to avoid:** Rewrite `__init__.py` before `python -m build --wheel`, then `git checkout -- llama_cpp/__init__.py` after build. This is safe because the build reads the file at configure time; the restore happens after the wheel is produced.
**Warning signs:** Wheel filename shows `0.3.20` without the `+cu126.ll<sha>` local segment.

### Pitfall 6: hashFiles with Submodule .git Path

**What goes wrong:** `hashFiles('vendor/llama.cpp/.git/HEAD')` may fail or produce inconsistent results because submodule `.git` can be either a file (git-dir indirection) or a directory.
**Why it happens:** Git submodule internals differ between regular clone and `actions/checkout` submodule handling.
**How to avoid:** Hash source files instead: `hashFiles('vendor/llama.cpp/CMakeLists.txt', 'vendor/llama.cpp/src/**')`. This also better reflects actual compilation dependency.
**Warning signs:** Cache key never matches, even when source hasn't changed.

### Pitfall 7: sccache CUDA Compilation on Windows SSL Issues

**What goes wrong:** The hendrikmuhs/ccache-action may fail to download sccache on Windows due to CRYPT_E_REVOCATION_OFFLINE curl errors.
**Why it happens:** Windows certificate revocation checking can fail intermittently on GitHub Actions runners.
**How to avoid:** Use `install: binary` option on the action (pre-built binary download), or pin a known-good sccache version. The action's default `install: detect` should handle this, but monitor for failures.
**Warning signs:** sccache setup step fails with SSL/TLS errors.

## Code Examples

### Complete Build Step (BLD-01, BLD-03, BLD-04, BLD-08, BLD-12)

```powershell
# Source: derived from upstream build-wheels-cuda.yaml + Phase 1 STATE.md constraints
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

# ---- vcvarsall activation ----
$installPath = '${{ steps.probe-msvc.outputs.install_path }}'
$selectedFull = '${{ steps.probe-msvc.outputs.selected_full_version }}'
$vcvarsall = Join-Path $installPath 'VC\Auxiliary\Build\vcvarsall.bat'

# Capture environment diff from vcvarsall
$envBefore = @{}
Get-ChildItem env: | ForEach-Object { $envBefore[$_.Name] = $_.Value }
$envLines = cmd /c "`"$vcvarsall`" x64 -vcvars_ver=$selectedFull >nul 2>&1 && set"
foreach ($line in $envLines) {
  if ($line -match '^([^=]+)=(.*)$') {
    $name = $Matches[1]; $value = $Matches[2]
    if (-not $envBefore.ContainsKey($name) -or $envBefore[$name] -ne $value) {
      Set-Item "env:$name" $value
    }
  }
}
Write-Host "vcvarsall activated: $(Get-Command cl.exe -ErrorAction Stop | Select-Object -ExpandProperty Source)"

# ---- CUDA paths ----
$env:CUDA_PATH = "$env:CONDA_PREFIX\Library"
$env:CUDA_HOME = "$env:CONDA_PREFIX\Library"

# ---- Version override ----
$initFile = 'llama_cpp/__init__.py'
$content = Get-Content $initFile -Raw
$verMatch = [regex]::Match($content, '__version__\s*=\s*"([^"]+)"')
if (-not $verMatch.Success) { throw "Cannot parse version from $initFile" }
$baseVersion = $verMatch.Groups[1].Value
$llamaSha = '${{ steps.assert-submodule.outputs.llama_sha }}'
# Ensure 12-char SHA
$llamaSha12 = $llamaSha.Substring(0, [Math]::Min(12, $llamaSha.Length))
$fullVersion = "$baseVersion+cu126.ll$llamaSha12"
Write-Host "Wheel version: $fullVersion"

$newContent = $content -replace ('__version__\s*=\s*"[^"]+"'), "__version__ = `"$fullVersion`""
Set-Content $initFile -Value $newContent -NoNewline

# ---- CMAKE_ARGS ----
$cudaArchs = '80-real;86-real;89-real;90-real;90-virtual'
$env:CMAKE_ARGS = @(
  '-G Ninja'
  '-DGGML_CUDA=on'
  "-DCMAKE_CUDA_ARCHITECTURES=$cudaArchs"
  '-DGGML_CUDA_FORCE_MMQ=ON'
  '-DGGML_AVX2=off'
  '-DGGML_FMA=off'
  '-DGGML_F16C=off'
  '-DCMAKE_C_COMPILER_LAUNCHER=sccache'
  '-DCMAKE_CXX_COMPILER_LAUNCHER=sccache'
  '-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache'
  '-DCMAKE_POLICY_DEFAULT_CMP0141=NEW'
  '-DCMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded'
) -join ' '

$env:CMAKE_BUILD_PARALLEL_LEVEL = '2'
$env:SKBUILD_WHEEL_PY_API = ''  # Override py3 -> cpXX tag

# ---- Build ----
python -m build --wheel
if ($LASTEXITCODE -ne 0) { throw "python -m build --wheel failed with exit code $LASTEXITCODE" }

# ---- Restore __init__.py ----
git checkout -- $initFile
```

### Wheel Tag Assertion (BLD-10)

```powershell
# Source: BLD-10 requirement + PEP 427 wheel filename spec
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$pyVer = '${{ inputs.python_version }}' -replace '\.', ''  # '3.11' -> '311'
$expectedPattern = "llama_cpp_python-.*-cp${pyVer}-cp${pyVer}-win_amd64\.whl$"

$wheels = @(Get-ChildItem dist/*.whl)
if ($wheels.Count -ne 1) {
  throw "Expected exactly 1 wheel in dist/, found $($wheels.Count): $($wheels.Name -join ', ')"
}
$wheelName = $wheels[0].Name

if ($wheelName -notmatch $expectedPattern) {
  $msg = "Wheel tag assertion FAILED.`n" +
         "  Expected pattern: $expectedPattern`n" +
         "  Actual filename:  $wheelName`n" +
         "`n" +
         "REMEDIATION:`n" +
         "  - If tag is 'py3-none': set SKBUILD_WHEEL_PY_API='' (empty string) before build`n" +
         "  - If tag is 'abi3': check pyproject.toml wheel.py-api setting`n" +
         "  - If tag is 'none-any': a platform-specific library is missing from the wheel"
  throw $msg
}
Write-Host "OK: wheel tag matches $expectedPattern -- $wheelName"
```

### Wheel Size Assertion (BLD-11)

```powershell
# Source: BLD-11 requirement
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$maxSizeMB = 400
$wheel = (Get-ChildItem dist/*.whl | Select-Object -First 1)
$sizeMB = [math]::Round($wheel.Length / 1MB, 2)

if ($sizeMB -ge $maxSizeMB) {
  $msg = "Wheel size assertion FAILED: $sizeMB MB >= $maxSizeMB MB limit.`n" +
         "  File: $($wheel.Name)`n" +
         "`n" +
         "REMEDIATION:`n" +
         "  - Reduce CMAKE_CUDA_ARCHITECTURES (currently 5 targets)`n" +
         "  - Drop a real target (e.g. 80-real if Ampere is not needed)`n" +
         "  - Check if debug symbols are accidentally embedded (should be /Z7 for sccache only)"
  throw $msg
}
Write-Host "OK: wheel size $sizeMB MB < $maxSizeMB MB -- $($wheel.Name)"
```

### sccache Stats (BLD-02, BLD-03)

```powershell
# Run after build to verify cache is working
sccache --show-stats
# On a rebuild (cache warm), expect hit rate > 30%
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `actions/cache@v3` unified | `actions/cache/restore@v4` + `actions/cache/save@v4` split | 2024 | Enables if: always() on save step |
| `/Zi` default debug info | CMP0141 + `/Z7` embedded | CMake 3.25 (2022) | Required for sccache to work with MSVC |
| ilammy/msvc-dev-cmd | vcvarsall.bat in pwsh subprocess | Phase 1 discovery | Avoids INCLUDE/LIB overflow with conda |
| `SETUPTOOLS_SCM_PRETEND_VERSION` | `__init__.py` rewrite + git restore | Phase 2 discovery | scikit-build-core regex provider ignores setuptools_scm |
| `wheel.py-api = "py3"` from pyproject.toml | `SKBUILD_WHEEL_PY_API=""` override | Phase 2 discovery | Gets correct cpXX wheel tag |

**Deprecated/outdated:**
- `Jimver/cuda-toolkit` action: Hangs on windows-2022 + CUDA 12.5+. Replaced by mamba install (TC-05).
- `ilammy/msvc-dev-cmd`: Causes vcvarsall overflow. Replaced by direct vcvarsall call (Phase 1).
- `CUDA_PATH` env var: Not set by mamba. Use `$env:CONDA_PREFIX\Library` (Phase 1).

## Open Questions

1. **SKBUILD_WHEEL_PY_API empty string behavior**
   - What we know: scikit-build-core's config system maps all options to SKBUILD_* env vars. Setting `wheel.py-api` to empty string in pyproject.toml uses the default Python version tag.
   - What's unclear: Does `SKBUILD_WHEEL_PY_API=""` (empty string env var) properly override the pyproject.toml setting? Or does it need `-C wheel.py-api=""` config-setting?
   - Recommendation: Try `SKBUILD_WHEEL_PY_API=""` first. If it produces `py3-none`, fall back to `-C wheel.py-api=""` config-setting. If that also fails, use `SKBUILD_WHEEL_PY_API=cp311`. Verify empirically on first dispatch.
   - **Confidence:** MEDIUM

2. **CMP0141 effectiveness with scikit-build-core**
   - What we know: sccache#2244 reports the policy sometimes doesn't convert /Zi to /Z7. The policy must be set before project(). scikit-build-core's internal CMake invocation may or may not honor CMAKE_POLICY_DEFAULT_CMP0141 from CMAKE_ARGS.
   - What's unclear: Whether scikit-build-core passes the policy variable early enough in its cmake invocation.
   - Recommendation: Include CMP0141 flags in CMAKE_ARGS AND check sccache stats after first build. If hit rate is near zero, add explicit `/Z7` via `CMAKE_C_FLAGS_INIT="/Z7"` + `CMAKE_CXX_FLAGS_INIT="/Z7"`.
   - **Confidence:** MEDIUM

3. **sccache + nvcc + Ninja on Windows interaction**
   - What we know: sccache supports nvcc. CMAKE_CUDA_COMPILER_LAUNCHER=sccache is the standard approach. Ninja works with CUDA on Windows.
   - What's unclear: Whether there are edge cases with sccache wrapping nvcc when nvcc itself needs to invoke cl.exe as host compiler on Windows.
   - Recommendation: Set `CMAKE_CUDA_HOST_COMPILER` to the full path of cl.exe (from vcvarsall activation) to avoid ambiguity.
   - **Confidence:** MEDIUM

4. **Mamba pkgs directory path on Windows**
   - What we know: The mamba pkgs cache needs a consistent path. MAMBA_ROOT_PREFIX is set by setup-miniconda.
   - What's unclear: Exact path to cache (could be `${{ env.MAMBA_ROOT_PREFIX }}/pkgs` or `${{ env.CONDA }}/pkgs`).
   - Recommendation: Log the path in a diagnostic step, then use `${{ env.MAMBA_ROOT_PREFIX }}/pkgs`. Verify on first dispatch.
   - **Confidence:** MEDIUM

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | GitHub Actions workflow dispatch + PowerShell assertions (same as Phase 1) |
| Config file | `.github/workflows/build-wheels-cuda-windows.yaml` (self-contained) |
| Quick run command | `actionlint .github/workflows/build-wheels-cuda-windows.yaml` |
| Full suite command | `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.6.3 && gh run watch --exit-status` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BLD-01 | Wheel produced via python -m build | dispatch | `gh run watch --exit-status` (build step green) | Wave 0 |
| BLD-02 | sccache setup via ccache-action | dispatch | Check step green + `sccache --show-stats` in log | Wave 0 |
| BLD-03 | sccache wired to CMake launchers | dispatch | grep CMAKE_ARGS for COMPILER_LAUNCHER in workflow | Wave 0 |
| BLD-04 | /Z7 embedded debug via CMP0141 | dispatch | sccache stats show >0 cache hits on rebuild | Wave 0 |
| BLD-05 | CUDA cache (mamba pkgs) | dispatch | Fulfilled by BLD-06 per CONTEXT.md | Wave 0 |
| BLD-06 | Mamba cache with if: always() | dispatch | Cache restore/save steps in log + retry build shows restore | Wave 0 |
| BLD-07 | VS integration redo each time | dispatch | BuildCustomizations copy step green | Wave 0 |
| BLD-08 | Ninja + parallel level | dispatch | CMAKE_ARGS contains `-G Ninja` + CMAKE_BUILD_PARALLEL_LEVEL logged | Wave 0 |
| BLD-09 | Per-step timeout-minutes | lint | `grep -c 'timeout-minutes' workflow` >= expected count | Wave 0 |
| BLD-10 | Wheel tag assertion cp311-cp311 | dispatch | Tag assertion step green (regex match) | Wave 0 |
| BLD-11 | Wheel size < 400 MB | dispatch | Size assertion step green | Wave 0 |
| BLD-12 | Version embeds SHA | dispatch | Wheel filename contains `+cu126.ll` segment | Wave 0 |
| BLD-13 | Wheel uploaded as artifact | dispatch | upload-artifact step green | Wave 0 |

### Sampling Rate
- **Per task commit:** `actionlint .github/workflows/build-wheels-cuda-windows.yaml`
- **Per wave merge:** Full dispatch + `gh run watch --exit-status`
- **Phase gate:** Full suite green before `/gsd:verify-work`; sccache stats showing >30% hit rate on a second dispatch (warm cache)

### Wave 0 Gaps
- None -- existing Phase 1 workflow infrastructure provides the substrate. Phase 2 appends steps to the existing preflight job (or adds a new build job with `needs: [preflight]`).

## Recommended Timeout Values (BLD-09)

| Step | Recommended timeout-minutes | Rationale |
|------|---------------------------|-----------|
| Copy VS BuildCustomizations | 2 | File copy, < 5 seconds expected |
| Restore mamba cache | 5 | Network I/O for cache download |
| Install build deps (pip) | 5 | Small packages (build, wheel, ninja) |
| sccache setup | 5 | Binary download + config |
| Build wheel (vcvarsall + cmake + ninja) | 60 | Cold CUDA build with 5 arch targets on 4-vCPU runner |
| Assert wheel tag | 2 | Instant file check |
| Assert wheel size | 2 | Instant file check |
| Upload artifact | 10 | Up to ~400 MB upload |
| Save mamba cache | 5 | Network I/O |
| sccache stats | 2 | Instant |

**Total budget:** ~98 minutes for build steps. Well within the 6-hour runner limit. The build step itself (60 min) is the dominant cost; warm builds with sccache should be 15-30 min.

## Sources

### Primary (HIGH confidence)
- Existing workflow file `.github/workflows/build-wheels-cuda-windows.yaml` -- Phase 1 output, probe-msvc outputs
- Existing `pyproject.toml` -- scikit-build-core config, wheel.py-api = "py3", regex version provider
- Existing `llama_cpp/__init__.py` -- version `0.3.20`, regex provider input
- Existing upstream `build-wheels-cuda.yaml` -- CMAKE_ARGS pattern, python -m build invocation
- Phase 1 STATE.md -- vcvarsall overflow discovery, CONDA_PREFIX for CUDA paths, mamba pwsh requirement

### Secondary (MEDIUM confidence)
- [scikit-build-core configuration docs](https://scikit-build-core.readthedocs.io/en/stable/configuration/) -- SKBUILD_* env vars, wheel.py-api, config-settings
- [hendrikmuhs/ccache-action](https://github.com/hendrikmuhs/ccache-action) -- action.yml inputs, sccache variant for Windows
- [actions/cache README](https://github.com/actions/cache) -- restore/save split pattern, if: always() for save-on-failure
- [mozilla/sccache](https://github.com/mozilla/sccache) -- CUDA/nvcc support, /Z7 requirement for MSVC
- [sccache#2244](https://github.com/mozilla/sccache/issues/2244) -- CMP0141 known issue with /Z7 conversion
- [GitHub community discussion on vcvarsall](https://github.com/orgs/community/discussions/26887) -- env vars don't persist across steps
- [Upstream llama-cpp-python wheel index](https://abetlen.github.io/llama-cpp-python/whl/cpu/llama-cpp-python/) -- wheels use cp311-cp311 tags (not py3-none)
- [boneylizardwizard build guide](https://huggingface.co/boneylizardwizard/llama-cpp-python-038-cu128-gemma3-wheel/blob/main/Build.Guide.md) -- SCB_LOCAL_VERSION env var reference
- [ggerganov/llama.cpp PR #11516](https://github.com/ggerganov/llama.cpp/pull/11516) -- ccache/sccache CI integration, Windows issues

### Tertiary (LOW confidence)
- SKBUILD_WHEEL_PY_API empty string override behavior -- inferred from docs pattern, not empirically verified
- CMAKE_CUDA_COMPILER_LAUNCHER + sccache + nvcc + Windows interaction -- documented but edge cases unclear
- Mamba pkgs cache path on Windows runners -- needs empirical verification

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- using same tools as upstream + well-documented Actions
- Architecture: MEDIUM-HIGH -- vcvarsall pattern is well-understood, but CUDA+sccache+Ninja combo on Windows has edge cases
- Version override: MEDIUM -- the __init__.py rewrite approach is reliable, but the SKBUILD_WHEEL_PY_API override needs empirical validation
- Cache strategy: HIGH -- actions/cache split pattern is well-documented and widely used
- Pitfalls: HIGH -- catalogued from Phase 1 discoveries + official issue trackers

**Research date:** 2026-04-16
**Valid until:** 2026-05-16 (30 days -- stable domain, tools are mature)
