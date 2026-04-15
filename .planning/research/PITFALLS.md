# Pitfalls Research — Windows CUDA Wheel Build CI

**Domain:** Windows x64 CUDA Python wheel build pipeline in GitHub Actions with gh-pages pip index publishing
**Researched:** 2026-04-15
**Confidence:** HIGH (critical path: VS/nvcc matrix, `-allow-unsupported-compiler` segfault, CPU-only smoke-test feasibility) · MEDIUM (wheel stripping specifics) · LOW (exact current VS 17.x on the runner today — image churn makes this inherently time-sensitive)

> **Scope note.** This document does not repeat generic CI hygiene. Every pitfall listed here has either *(a)* caused a documented upstream failure on `abetlen/llama-cpp-python` CI, or *(b)* is a structural trap in the Windows + CUDA + Python + GitHub Actions + gh-pages stack that the user spec has not yet explicitly planned for. The two big known ones (VS/nvcc rotation, `-allow-unsupported-compiler` segfault) are cross-referenced upstream and are the *context*, not the whole story.

---

## Critical Pitfalls

### Pitfall 1: `-allow-unsupported-compiler` produces wheels that install but segfault

**What goes wrong:**
Build succeeds, `pip install` succeeds, `import llama_cpp` succeeds, then the first `Llama(model_path=...)` call segfaults inside the CUDA kernel path. No stack trace. No error message. The published wheel looks healthy in every automated check except "does it actually run."

**Why it happens:**
`nvcc` refuses to compile against MSVC versions it has not been tested with, emitting:

> `fatal error C1189: #error: -- unsupported Microsoft Visual Studio version! Only the versions between 2017 and 2022 (inclusive) are supported!`

The flag `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` silences this. It does *not* fix the underlying ABI/intrinsic mismatch. `nvcc` compiles device code by rewriting it through the MSVC preprocessor + host headers; when those headers expose intrinsics or templates the CUDA toolchain wasn't validated against, you get corrupted object code that passes link-time checks but fails at runtime. The current `.github/workflows/build-wheels-cuda.yaml` line 159 **unconditionally sets this flag**, even on the Linux path — so this trap is already baked into the workflow we're inheriting.

**How to avoid:**
1. **Remove `--allow-unsupported-compiler` from `CMAKE_ARGS`.** Let the build fail loudly when VS/nvcc don't match. That is the signal.
2. **Pin VS via `setup-msbuild`** to a range validated against nvcc 12.4.1. Based on CUDA 12.4 release notes and NVIDIA's host-config header shipped in that toolkit, the VS 2022 range `[17.0, 17.10)` is the safest broad pin; `[17.9, 17.10)` is narrower and preferred. MSVC 19.40 (VS 17.10) was the first MSVC version added to CUDA 12.4's whitelist but arrived mid-cycle and triggered edge-case breakage in adjacent projects (cupy #8578, nerfstudio #3171).
3. **Gate publishing on the smoke test** that actually instantiates `Llama(...)` and generates at least one token. The spec already calls this out in §9 — enforce it as a *required* job-level dependency before `action-gh-release` / gh-pages push.

**Warning signs:**
- In CI logs: `--allow-unsupported-compiler` appears anywhere in the `CMAKE_ARGS` or `nvcc` command line.
- Build step prints `unsupported Microsoft Visual Studio version` but proceeds anyway.
- Smoke test step is marked `continue-on-error: true`, or runs after `action-gh-release`.
- Wheel size is "normal" (~200–400 MB) but the workflow had compiler warnings flagged as "unsupported".

**Phase to address:** Phase 1 (workflow scaffold + VS pinning). Must never be deferred. Cross-ref [upstream #1543](https://github.com/abetlen/llama-cpp-python/issues/1543) — reporter confirmed: *"while that did allow the wheel to be successfully compiled, it crashed on my Windows tests."*

---

### Pitfall 2: windows-2022 runner VS 17.x rolls forward out from under the build

**What goes wrong:**
The workflow worked yesterday; this morning it fails with `unsupported Microsoft Visual Studio version` or with a stranger error (`_MSC_VER` doesn't match any branch in `crt/host_config.h`, MSVC STL templates fail to compile inside CUDA headers — see [runner-images #10575](https://github.com/actions/runner-images/issues/10575)). Nobody changed our workflow. The GitHub-hosted `windows-2022` image was updated during the night to a newer VS 17.x.

**Why it happens:**
`windows-2022` is a **moving image**, not a version pin. Microsoft ships new VS 17.x every ~4–8 weeks and the actions/runner-images repo rolls it in within 1–2 weeks. The CUDA toolkit's `crt/host_config.h` ships with a hard-coded `_MSC_VER` allow-list; anything newer is rejected. CUDA 12.4 whitelisted MSVC 19.40 (VS 17.10); VS 17.11/17.12 shipped MSVC 19.41+, which CUDA 12.4.1 did **not** add. Every image rotation is a coin flip on whether the currently-installed VS exceeds our nvcc's whitelist. This is the exact ambient cause of the historical upstream breakage series (#1543, #1551, #1838, #1894) and the direct reason for commit `98fda8c` disabling Windows CUDA CI in July 2025.

**How to avoid:**
1. **Do not rely on the default MSVC on the runner.** Always invoke `microsoft/setup-msbuild@v2` with an explicit `vs-version: '[17.0,17.10)'` (or tighter) to *select* an acceptable VS from the images that still ship one in range. Note: if the image no longer ships any VS in the pinned range, `setup-msbuild` fails — that failure is actually useful, it tells you the image rotated past your pin.
2. **Also pin `CC` / `CXX` to a specific MSVC toolset directory**, not just the VS version. VS 17.9 still ships VS 17.10 and 17.11 MSVC toolsets side-by-side via `vcvarsall.bat --vcvars_ver=14.39` (or specific version). Use `$env:VCToolsVersion` or explicitly invoke `vcvarsall.bat 14.39` before CMake. Just having VS 17.9 installed does not guarantee `cl.exe` is 19.39.
3. **Add a CI preflight step** that prints `cl.exe /Bv` and `nvcc --version` *and fails* if the MSVC major.minor is above a hard ceiling. Example:
   ```powershell
   $cl_ver = (& cl.exe 2>&1 | Select-String 'Version ([0-9]+\.[0-9]+)').Matches[0].Groups[1].Value
   if ([version]$cl_ver -ge [version]'19.40') {
       throw "MSVC $cl_ver is newer than CUDA 12.4.1's whitelist ceiling (19.40 allowed only with patch; 19.41+ rejected). Pin vs-version tighter or bump CUDA."
   }
   ```
   This flips silent image rotation into a loud, specific, actionable failure.
4. **Consider `windows-2025` as a fallback runner** if `windows-2022` rotates past 17.10. VS 2026 is showing up on runner-images (#13291, #13272, #13512) — once that lands on `windows-2025`, the whole matrix shifts again and we need to revisit.
5. **Schedule a weekly "canary" workflow run** (`schedule: '0 6 * * 1'`) that builds with current pins but doesn't publish. If the canary goes red Monday, we know before a real release goes out.

**Warning signs:**
- CI logs show `Microsoft (R) C/C++ Optimizing Compiler Version 19.41.*` or higher.
- `setup-msbuild` step silently picks a newer VS than the range you requested (bug in action when action-versioned: use `@v2`, not `@main`).
- Warnings like `warning: option '--unsupported-compiler' is ineffective without ...` appearing unexpectedly.
- Last green run was more than 2 weeks ago on a `workflow_dispatch`-only workflow (image has almost certainly rotated under you).

**Phase to address:** Phase 1 (initial workflow) + Phase 3 or later (ongoing hardening: add canary schedule + MSVC-version assertion step). Cross-ref [runner-images #10448 (17.11)](https://github.com/actions/runner-images/issues/10448), [#10954 (17.12)](https://github.com/actions/runner-images/issues/10954), [cupy/cupy#8578](https://github.com/cupy/cupy/issues/8578), [nerfstudio #3171](https://github.com/nerfstudio-project/nerfstudio/issues/3171).

---

### Pitfall 3: `CUDA_PATH` mis-set — mamba toolkit vs Jimver installer extraction race

**What goes wrong:**
`nvcc` runs, CMake's `find_package(CUDAToolkit)` finds the toolkit, the build completes — but either (a) the detected CUDA version tag in the wheel filename is wrong (e.g. wheel labelled `cu121` when we targeted `12.4.1`), or (b) the build links against the *Jimver-extracted* CUDA libs while nvcc was *mamba's*, leading to an ABI mismatch that doesn't fire until `Llama(...)` hits a cuBLAS routine at runtime.

**Why it happens:**
The existing workflow installs CUDA **two different ways in two different steps**:
- Step "Get Visual Studio Integration" (lines 78–86): downloads the **full Jimver** zip just to extract `MSBuildExtensions/` (VS integration files).
- Step "Install Dependencies" (lines 96–113): `mamba install cuda-toolkit=$CUDAVER` installs an **entirely separate** CUDA into `$CONDA_PREFIX`.

Then the Build Wheel step (lines 115+) sets `CUDA_PATH=$CONDA_PREFIX` — pointing at mamba's install — but also overwrites `$env:GITHUB_ENV` with `CUDA_PATH_V12_4=$env:CONDA_PREFIX` (line 93–94), polluting later steps. If any CMake find logic traverses `CUDA_PATH_VX_Y` vars first (which `FindCUDAToolkit.cmake` does in some paths), the wrong root may win. Additionally, `mamba install cuda-toolkit` from the `nvidia/label/cuda-$VER` channel historically has **version mismatches** between the nvcc binary and the cudart headers: on the Linux branch the workflow explicitly pins `cuda-nvcc_linux-64`, `cuda-cudart`, `cuda-cudart-dev` to the same label (line 106), but the Windows branch does **not** (line 108) — meaning on Windows we can get nvcc 12.4.1 with cudart 12.1 headers if the channel solve is loose.

**How to avoid:**
1. **Pick one installation path, not two.** Either:
   - *Option A (recommended):* install everything via Jimver (`Jimver/cuda-toolkit@v0.2.x`). This writes `CUDA_PATH` as a single source of truth and ships correct MSBuildExtensions in-place. Skip mamba entirely on Windows. Cleaner, faster, closer to how NVIDIA actually installs CUDA.
   - *Option B:* keep mamba, but do **not** extract MSBuildExtensions from a second Jimver zip. Mamba's cuda-toolkit package ships VS integration in `$CONDA_PREFIX/Library/extras/visual_studio_integration/...`. Point the VS BuildCustomizations copy at that path.
2. **Apply the Linux-style pin on Windows too:**
   ```powershell
   mamba install -y --channel-priority strict --override-channels -c $cudaChannel `
     "$cudaChannel::cuda-toolkit=$cudaVersion" `
     "$cudaChannel::cuda-nvcc=$cudaVersion" `
     "$cudaChannel::cuda-cudart=$cudaVersion" `
     "$cudaChannel::cuda-cudart-dev=$cudaVersion"
   ```
   Note: use `--channel-priority strict`, not `flexible`, so mamba can't pull a mismatched cudart from conda-forge.
3. **Explicitly unset every `CUDA_PATH_V*_*` variant** before the build step, then set exactly one `CUDA_PATH`, `CUDA_HOME`, `CUDAToolkit_ROOT` pointing at the single root.
4. **Assert the nvcc version matches the requested cuda version at the top of Build Wheel**, before running CMake:
   ```powershell
   $v = (& nvcc --version | Select-String 'release ([0-9]+\.[0-9]+)').Matches[0].Groups[1].Value
   if ($v -ne '12.4') { throw "nvcc reports $v but matrix requested $env:CUDAVER" }
   ```
   The workflow at lines 150–153 already extracts this version but only uses it for the *wheel tag*, not for validation.

**Warning signs:**
- Wheel filename ends with a different `cuXYZ` than the matrix requested (line 171 `CUDA_VERSION=$cudaTagVersion` uses the *detected* version, so if it drifts, the filename drifts silently).
- `where nvcc` returns multiple paths in CI.
- `$env:CUDA_PATH_V12_4` set to one prefix but `$env:CUDA_PATH` set to another.
- CMake log shows `-- Found CUDAToolkit: ... (found version "12.1.1")` when you requested 12.4.1.

**Phase to address:** Phase 1 (workflow scaffold) — get this right the first time, or you'll be chasing ABI ghosts for days.

---

### Pitfall 4: MSBuildExtensions copied to VS 2019 path — silently no-ops on windows-2022

**What goes wrong:**
Line 92 of the current workflow:
```powershell
(gi 'C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\MSBuild\Microsoft\VC\*\BuildCustomizations').fullname.foreach({cp $y $_})
```
On `windows-2022`, `\2019\Enterprise\` **does not exist** (only `\2022\Enterprise\`). `Get-Item` with a wildcard path that matches zero items returns `$null`. `$null.fullname.foreach(...)` throws `InvalidOperation: You cannot call a method on a null-valued expression` **in strict mode** but in default PowerShell it silently treats `$null.fullname` as `$null`, and `foreach` on `$null` is a no-op. The CUDA .props/.targets are never copied. MSBuild-based CUDA projects then fail to find CUDA rules — *but `scikit-build-core` doesn't use MSBuild*, it calls CMake directly with the Ninja or NMake generator. So this copy step is *mostly* cargo-culted from the jllllll workflow which *did* use MSBuild, and CMake will build fine without it on Windows. The danger: someone sees the step "succeed" and assumes integration is set up, then later adds a diagnostic `msbuild` call or a CUDA `.vcxproj` target and hits a confusing failure.

**Why it happens:**
1. Path string is hardcoded to the VS 2019 install layout that only existed on the deprecated `windows-2019` runner.
2. PowerShell's permissive null handling turns a "path does not exist" bug into silent success.
3. The whole step is arguably unnecessary for a CMake + Ninja build, but was inherited whole-cloth from upstream.

**How to avoid:**
1. **Detect VS install dynamically**, don't hardcode:
   ```powershell
   $vsRoots = Get-ChildItem 'C:\Program Files\Microsoft Visual Studio\2022\*' -ErrorAction Stop
   # Enterprise, Community, BuildTools — whichever the runner has
   $bc = $vsRoots | ForEach-Object { Get-ChildItem "$($_.FullName)\MSBuild\Microsoft\VC\*\BuildCustomizations" -ErrorAction SilentlyContinue }
   if (-not $bc) { throw 'No VS 2022 BuildCustomizations path found — runner image layout changed.' }
   $bc | ForEach-Object { Copy-Item "$msbuildExtensionsSrc\*" $_ }
   ```
   Note the path on `windows-2022` is `C:\Program Files\...` (no `(x86)`), because VS 2022 is the first 64-bit VS. The old x86 path was VS 2019 and earlier.
2. **Evaluate whether the step is needed at all.** If the build is 100% CMake + Ninja (which `python -m build --wheel` with scikit-build-core will do), drop the MSBuildExtensions copy entirely. It's only needed when MSBuild consumes `.vcxproj` files that `#import` the CUDA .props.
3. **If you keep it, make path-not-found fatal:**
   ```powershell
   Set-StrictMode -Version Latest
   $ErrorActionPreference = 'Stop'
   ```
   at the top of the step. This flips the silent no-op into a loud "path doesn't exist" failure.

**Warning signs:**
- `Install Visual Studio Integration` step finishes in under 1 second (real copies take 2–10s; a no-op finishes instantly).
- No lines in the step log starting with `Copying` or showing file paths.
- Same workflow "works" whether the preceding MSBuildExtensions extraction step ran or was skipped (proves the copied files weren't used).

**Phase to address:** Phase 1 (fix when adapting to VS 2022 paths). User spec §5 already flags the 2019→2022 path issue — make sure the fix uses dynamic detection, not just a find-and-replace of `2019` → `2022`.

---

### Pitfall 5: sccache + MSVC + CUDA — cache miss every run if debug flags aren't forced to `/Z7`

**What goes wrong:**
sccache is wired in, the cache action reports a successful restore, the build takes 30–40 minutes anyway, and the sccache stats at end-of-step show 0% hit rate. Or worse: the cache restores but CMake still emits `/Zi`, sccache refuses to cache those translation units, the cache fills with only the handful of files that happened to use `/Z7` or no debug info, and every subsequent run re-compiles 95% of the CUDA kernels from scratch. Build budget blown.

**Why it happens:**
sccache has documented issues with MSVC's default debug info mode (`/Zi`): multiple TUs write to the same `.pdb` file, which sccache cannot safely cache because the PDB is a *shared, mutable* output. sccache's MSVC adapter requires `/Z7` (debug info embedded in the `.obj`) for cacheability. CMake's default on Windows for a `Release` config is `/Zi` in many project templates; for `RelWithDebInfo` it's definitely `/Zi`. Additionally, nvcc wraps MSVC for host compilation and passes through whatever `-Xcompiler` flags you give it — if your `CMAKE_CXX_FLAGS` has `/Zi`, your CUDA host compilation inherits it, and those CUDA objects also miss the cache.

sccache + nvcc is supported in recent sccache (confirmed in 0.10+, current 2026 builds), but each nvcc invocation creates temp files in `$TMP` and sccache hashes the command line + preprocessed source. If your CMake invocation embeds absolute build-directory paths in `-I` flags, the hash changes every build (different runner = different temp path), and you get cache misses by path alone.

**How to avoid:**
1. **Force `/Z7` for MSVC (and for CUDA host compiler):**
   ```
   -DCMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded
   -DCMAKE_POLICY_DEFAULT_CMP0141=NEW
   ```
   (CMP0141 is required to make the abstract `MSVC_DEBUG_INFORMATION_FORMAT` property actually take effect.) Alternatively, scrub flags explicitly:
   ```
   -DCMAKE_CXX_FLAGS_RELEASE="/MD /O2 /Ob2 /DNDEBUG /Z7"
   -DCMAKE_C_FLAGS_RELEASE="/MD /O2 /Ob2 /DNDEBUG /Z7"
   -DCMAKE_CUDA_FLAGS="-Xcompiler=/Z7"
   ```
2. **For Release wheels, strip debug info entirely** (see Pitfall 10). `/Z7` is primarily about cache-ability; if the final wheel is stripped, you don't need the embedded debug info anyway.
3. **Make the sccache cache key stable across runs.** `hendrikmuhs/ccache-action@v1.2` with `variant: sccache` and `append-timestamp: false` is correct (the spec already has this). Also set `SCCACHE_IGNORE_SERVER_IO_ERROR=1` and `SCCACHE_DIRECT=true` to avoid preprocessor-path-sensitivity on CUDA files.
4. **Verify cache-ability explicitly after the build:**
   ```powershell
   & sccache --show-stats
   # Assert: "Compile requests executed" - "Cache hits" should be small after the first warm run.
   ```
   Fail the job (or at least emit a warning annotation) if the hit rate on rebuild is below, say, 30%. This catches silent cache-invalidation bugs before they burn budget.
5. **Realistic hit rate expectations:** 80–95% is achievable on a *pure source rebuild* (no CUDA arch flag changes, no toolkit version bump). Expect 0% on first run, 0% after a CUDA toolkit version change, 0% after a submodule bump of llama.cpp's ggml-cuda kernels if the included .cu files changed. For the typical "fix a YAML typo and re-dispatch" loop, realistic hit rate is 60–85%. The spec's "5–10 min warm" target is achievable **only** if /Z7 is forced and cache keys are stable.

**Warning signs:**
- `sccache --show-stats` reports `Non-cacheable compilations: N` where N > 0 on MSVC TUs.
- `Cache hits (C/C++)` stays at 0 on a rebuild where only non-CUDA files changed.
- Build time for "warm" rebuild is more than 50% of cold build time.
- sccache log (`SCCACHE_LOG=debug`) shows `Rejecting compilation due to unsupported argument: /Zi`.

**Phase to address:** Phase 2 (caching stack). Don't promise fast rebuilds in phase goals until /Z7 is verified; the keepassxc cache reference pattern in the spec will only work well once this is fixed.

---

### Pitfall 6: Wheel mis-tagged (abi3, wrong Python tag, generic platform)

**What goes wrong:**
Build produces `llama_cpp_python-0.3.20-cp39-abi3-win_amd64.whl` or `...-py3-none-any.whl` instead of the required `...-cp311-cp311-win_amd64.whl`. `pip install` on a 3.11 target will pick up the wheel happily if the tag is *compatible* (abi3 is compatible with cp311+), but the wheel built against Python 3.11 headers can crash when the actual Python runtime differs — and a `py3-none-any` tag is a lie, because the wheel contains a `.pyd` that is 100% platform-specific.

**Why it happens:**
scikit-build-core has an opinion about abi3. If `[tool.scikit-build]` has `wheel.py-api = "cp37"` or if `py-limited-api` is set in the C extension, you get abi3. llama-cpp-python's extension **is not Py_LIMITED_API-safe** (it uses ctypes wrappers, but also uses Python APIs that aren't in the stable ABI). So tagging abi3 is a bug, not a feature.

The `pyproject.toml` currently in the repo may or may not set this — worth auditing. xformers hit exactly this bug (#1270 — wheel built with 3.12 came out tagged `cp39-abi3` for all versions).

Additionally, `python -m build --wheel` runs `bdist_wheel` which *infers* the tag from the interpreter running it. If the runner's `python` at build time is the mamba-installed one (Python 3.11), but the earlier `setup-python@v5` step installed Python 3.12, you'll get a 3.12-tagged wheel running against a 3.11 interpreter — which segfaults on import.

**How to avoid:**
1. **Audit `pyproject.toml`**: ensure `[tool.scikit-build]` does **not** set `wheel.py-api`. If it does, override per-build with an env var or strip the section.
2. **Verify wheel tag before publish:**
   ```powershell
   $wheel = Get-Item dist\*.whl
   if ($wheel.Name -notmatch "cp$($pyverNoDot)-cp$($pyverNoDot)-win_amd64\.whl$") {
     throw "Wheel tag mismatch: $($wheel.Name) does not match cp$pyverNoDot-cp$pyverNoDot-win_amd64"
   }
   ```
   where `$pyverNoDot = "311"` for Python 3.11. This is a one-line gate that catches all three misconfigurations.
3. **Do not mix `setup-python` and `setup-miniconda` when the build runs Python.** Pick exactly one Python for the `python -m build` invocation and confirm with `python --version && where python` right before it.
4. **Do not use `--plat-name`** unless you know exactly what you're doing. Let scikit-build-core infer.

**Warning signs:**
- Wheel filename contains `abi3` or `none-any`.
- Wheel filename contains `cp312` but the matrix says 3.11.
- Two `python.exe` in `where python` output.

**Phase to address:** Phase 1 (build step) + Phase 3 (smoke test — the smoke test will catch a runtime segfault from tag mismatch, but better to fail earlier on the tag itself).

---

### Pitfall 7: Smoke test on CPU-only runner — does it actually prove the CUDA path works?

**What goes wrong (the real question in §10):**
The user spec §9 proposes running the smoke test on the Windows runner itself. The Windows runner has **no NVIDIA GPU and no NVIDIA driver installed**. Question: does importing `llama_cpp` with a CUDA-backend wheel, and calling `Llama(model_path=..., n_gpu_layers=0)`, actually exercise the CUDA DLLs enough to catch the `-allow-unsupported-compiler` corruption that is the whole reason we're doing this?

**Concrete answer (HIGH confidence):** **Partially yes, meaningfully.** Here's what actually happens:

1. `import llama_cpp` — loads `llama.dll` (the Python extension). This dll depends on `ggml.dll`, `ggml-cpu.dll`, and (in a CUDA build) `ggml-cuda.dll`. Windows resolves the import table at load time: **all three DLLs must be resolvable even if ggml-cuda is never *called*.** That means Windows must be able to find `cudart64_12.dll`, `cublas64_12.dll`, `cublasLt64_12.dll` on `PATH` or next to the .pyd. If the wheel doesn't bundle them and the runner doesn't have CUDA installed, `import llama_cpp` fails with `OSError: [WinError 126] The specified module could not be found` — that by itself is a useful signal.
2. **Once the DLL loads succeed**, `Llama(model_path='...')` runs ggml backend registration. In recent llama.cpp (post-October 2024), backends are *registered* at `llama_backend_init()`. The CUDA backend registration calls `cuInit()`, which in turn calls `cudaGetDeviceCount()`. **On a machine with no driver, this returns `cudaErrorNoDevice` (or `cudaErrorInsufficientDriver`), not a crash.** The backend is marked unavailable, ggml falls back to CPU. `Llama(...)` succeeds. 2 tokens get generated on CPU.
3. **What this catches:**
   - ✓ Wheel installation / tagging bugs (Pitfall 6).
   - ✓ DLL dependency bugs — missing cudart, missing cublas, wrong MSVC runtime (Pitfall 11 below).
   - ✓ Bad ABI in the *Python-facing layer* (ctypes struct layout, function signatures).
   - ✓ Bad code in the *CPU* ggml paths (which are also compiled by the same MSVC that compiled the CUDA host code).
   - ✓ Symbol-level corruption from `-allow-unsupported-compiler` *in the CPU TUs* (because the same broken MSVC/nvcc host compilation produces both CPU and CUDA host objects).
4. **What this does NOT catch:**
   - ✗ Broken CUDA *device code* (the .cu kernel .ptx/.cubin). If nvcc produced corrupt SASS for `sm_89`, the smoke test can't see it — the CUDA backend never loads.
   - ✗ CUDA context init bugs.
   - ✗ Wrong CUDA arch flags (wheel built without `sm_89` → silently runs CPU on an RTX 40xx user machine, works fine on the smoke test).

**How to avoid / augment:**
1. **Keep the CPU-only smoke test as the publish gate.** It catches the majority of "wheel compiles but is broken" cases (the `-allow-unsupported-compiler` trap manifests in the C++ host compilation, which the smoke test exercises).
2. **Add an additional verification that is CPU-runnable**: extract the wheel and inspect the SASS arches embedded in `ggml-cuda.dll`:
   ```powershell
   pip install llama_cpp_python-*.whl
   $cudadll = python -c "import llama_cpp, os; print(os.path.join(os.path.dirname(llama_cpp.__file__), 'lib', 'ggml-cuda.dll'))"
   # Use cuobjdump from the CUDA toolkit (it's on the runner in CONDA_PREFIX)
   & "$env:CONDA_PREFIX\bin\cuobjdump.exe" --list-elf $cudadll
   # Assert that each required arch in CMAKE_CUDA_ARCHITECTURES appears in the output.
   ```
   This turns a CPU-only runner into a reasonable sanity check that the wheel was actually built *for GPUs at all*, and with the right compute capabilities.
3. **Maintain a separate off-CI GPU smoke test** (manual, run on dev host before promoting a wheel from `pre-release` → `stable`). The spec already implies this ("le vrai test CUDA tu le fais en local de toute façon").
4. **For extra signal**, run the CPU-only smoke test *with* `n_gpu_layers=-1` and assert ggml logs a CUDA-backend-unavailable message. If it logs nothing about CUDA at all, the CUDA backend wasn't even linked. If it logs a *crash* trying to init CUDA (as opposed to "no devices available, falling back"), the runtime is broken.

**Warning signs:**
- Smoke test passes on CPU-only runner but users immediately report crashes → missing arch in `CMAKE_CUDA_ARCHITECTURES` or static-symbol corruption in device code.
- Smoke test *fails* with `WinError 126` → wheel is missing bundled CUDA DLLs (see Pitfall 11).
- Smoke test output shows zero references to "CUDA" or "cuBLAS" → CUDA backend not compiled in.

**Phase to address:** Phase 3 (smoke test). Document the limits of CPU-only smoke testing explicitly in the workflow comments — nobody should think "CI is green = wheel is good on a 4090."

---

### Pitfall 8: gh-pages race — two dispatch runs overwrite each other's wheels

**What goes wrong:**
A contributor kicks off a 3.11/cu124 build. Five minutes later they notice 3.12 is also needed and kick off a 3.12/cu124 build. Both finish around the same time. Both check out `gh-pages`, add their wheel to `whl/cu124/llama-cpp-python/`, regenerate `index.html`, commit, and push. The second push races the first: the action does either `force-push` (which erases the first wheel entirely) or `pull --rebase` (which, if `index.html` was regenerated with only the current wheel's entry, also erases the first wheel because it "wins" the rebase merge with only one file in the new state).

**Why it happens:**
- `peaceiris/actions-gh-pages@v4` default behavior on a non-empty `gh-pages` branch is a *merge* push, but the common pattern (including every example in the README) is to **regenerate the full publish directory each run**. If your publish dir only has `llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl`, and the branch already had both cp311 *and* cp312 wheels, the action pushes a tree with only cp311, wiping cp312.
- Without a `concurrency:` block, GitHub Actions runs both workflow dispatches simultaneously.
- `GITHUB_TOKEN`'s default `contents: write` is enough to push to `gh-pages`, but since both runs use the same token, there's no coarse mutex.

**How to avoid:**
1. **Use concurrency to serialize all gh-pages publishes:**
   ```yaml
   concurrency:
     group: gh-pages-publish
     cancel-in-progress: false
   ```
   **Not** `cancel-in-progress: true` — that would kill a half-finished build. Use `false` so the second run queues behind the first. One group name, workflow-wide, shared across *every* workflow that might touch `gh-pages`.
2. **Publish additively, not destructively.** Check out `gh-pages`, *preserve* existing files in `whl/cuXYZ/llama-cpp-python/`, add only the new wheel, regenerate `index.html` from `ls whl/cuXYZ/llama-cpp-python/*.whl`. Example with `peaceiris/actions-gh-pages`:
   ```yaml
   - uses: peaceiris/actions-gh-pages@v4
     with:
       github_token: ${{ secrets.GITHUB_TOKEN }}
       publish_dir: ./gh-pages-staging
       keep_files: true   # <-- critical: do NOT wipe files already on gh-pages
       destination_dir: whl/cu${{ env.CUDA_VERSION }}/llama-cpp-python
   ```
   `keep_files: true` is the default behavior in some action versions but is worth making explicit.
3. **Regenerate `index.html` server-side from the actual file listing**, not from the current run's output alone:
   ```yaml
   - name: Regenerate index
     run: |
       # After checking out gh-pages AND after adding new wheel
       python .ci/make_index.py whl/cu${{ env.CUDA_VERSION }}/llama-cpp-python/
   ```
   Where `make_index.py` globs `*.whl` and emits a PEP 503 `index.html`. Single source of truth: the filesystem.
4. **Use `GITHUB_TOKEN`, not a PAT, unless you need cross-repo.** `GITHUB_TOKEN` with `permissions: contents: write` is sufficient for pushing to `gh-pages` within the same repo and avoids PAT-rotation pain. PAT is only needed if you publish to a *different* repo (e.g., a dedicated `<user>/llama-cpp-python-windows-wheels` mirror).
5. **Do not use `force_orphan: true`** — this nukes the entire `gh-pages` history on every push, which does exactly the thing we're trying to prevent.

**Warning signs:**
- `gh-pages` branch has exactly one wheel in it after a publish (check `git log --oneline -- whl/cu124/` → each commit should *add* a wheel, not replace).
- Two workflow runs dispatched close together both show "success" but the user's `pip install --extra-index-url=...` only sees one version.
- Action logs show `force-push` or "discarded 1 commits".

**Phase to address:** Phase 4 (publishing). Required before the first multi-wheel publish; if only one wheel is ever published the bug is invisible.

---

### Pitfall 9: gh-pages / Fastly stale for ~10 minutes after push — debuggers think the wheel isn't there

**What goes wrong:**
Workflow succeeds, wheel is committed to `gh-pages`, `git show gh-pages:whl/cu124/llama-cpp-python/index.html` shows the new wheel. But `curl https://<user>.github.io/llama-cpp-python/whl/cu124/llama-cpp-python/` returns the *old* index. `pip install --extra-index-url=...` can't see the new wheel. Dev spends an hour debugging the workflow that worked fine.

**Why it happens:**
GitHub Pages is fronted by Fastly CDN. Default cache TTL on GH Pages content is 10 minutes; reported real-world stale windows go up to 30 minutes when Fastly edges are cold. `index.html` is aggressively cached because it has no unique hash in its URL. This is not a bug, it's by design — but it's invisible unless you've been bitten before.

**How to avoid:**
1. **After publishing, sleep + verify against `index.html` before claiming success:**
   ```yaml
   - name: Verify published index
     run: |
       $expected = "llama_cpp_python-${{ env.VERSION }}-cp${{ env.PYVER }}-cp${{ env.PYVER }}-win_amd64.whl"
       $url = "https://${{ github.repository_owner }}.github.io/llama-cpp-python/whl/cu${{ env.CUDA_VERSION }}/llama-cpp-python/"
       for ($i = 0; $i -lt 15; $i++) {
         $body = (Invoke-WebRequest -Uri $url -UseBasicParsing -Headers @{'Cache-Control'='no-cache'}).Content
         if ($body -match $expected) { Write-Host "Published: verified"; exit 0 }
         Start-Sleep -Seconds 60
       }
       throw "Published wheel not visible in Fastly after 15 minutes"
   ```
   This also catches failures of the previous pitfall (wheel wiped by race).
2. **Tell users upfront.** README should say: *"After a successful publish, the index may take up to 15 minutes to update due to CDN caching. If pip says 'No matching distribution', add `--no-cache-dir` and retry after a few minutes."*
3. **Include `--no-cache-dir` in the install documentation** — pip's own cache plus Fastly is a double miss window.
4. **Cannot manually purge Fastly** — you don't have the API key for GitHub's Fastly account. Waiting is the only option.

**Warning signs:**
- Workflow succeeded but `pip install --extra-index-url ...` says `No matching distribution`.
- Browser shows old `index.html` but GitHub's web UI of the `gh-pages` branch shows the new one.
- `curl -I <url>` shows `X-Cache: HIT` and an `Age:` header > 0.

**Phase to address:** Phase 4 (publishing) — bake the verification step into the workflow. Phase 5 (documentation) — user-facing note on the caching window.

---

### Pitfall 10: Wheel bloat — 500 MB+ with debug symbols and over-broad CUDA arches

**What goes wrong:**
Wheel ships at 600–800 MB. Blows past GitHub release asset limits (2 GB is the hard cap but 100 MB is the "normal file" cap without Git LFS). `pip install` times out on slow networks. `gh-pages` branch bloats (GitHub Pages has a soft 1 GB repo limit — not per-wheel, *total*). After 5 releases × 3 Python versions × 2 CUDA versions × 600 MB, you hit 18 GB and things start rejecting.

**Why it happens:**
- `/Z7` debug info (forced for sccache cacheability, Pitfall 5) bloats every `.obj` and gets linked into the final `.dll`. Default CMake Release on Windows ships with debug info in MSVC: `/Zi` + `/DEBUG`.
- CUDA device code: each `CMAKE_CUDA_ARCHITECTURES` entry (e.g. `70;75;80;86;89;90`) gets both an `sm_XY-real` SASS **and** a `compute_XY-virtual` PTX embedded. Six arches × ~30 MB of ggml-cuda kernels = 180 MB of device code alone. Embedded debug info (`-G`) multiplies by 5x.
- cuBLAS, cuBLASLt, cudart runtime DLLs bundled into the wheel: ~600 MB by themselves if naively copied from `$CUDA_PATH\bin\*.dll`.

The current workflow at line 159 already has partial mitigation: `-DCMAKE_CUDA_ARCHITECTURES=70-real;75-real;80-real;86-real;89-real;90-real;90-virtual` (real SASS, one virtual PTX for forward-compat). Good. But **`--allow-unsupported-compiler` is also on that line** (see Pitfall 1), and there's no explicit `strip` / `/DEBUG:NONE` / `dumpbin /PDBPATH` cleanup.

**How to avoid:**
1. **Build Release with debug info completely off for the final link:**
   ```
   -DCMAKE_BUILD_TYPE=Release
   -DCMAKE_MSVC_DEBUG_INFORMATION_FORMAT=""  # or "Embedded" + post-link strip
   -DCMAKE_EXE_LINKER_FLAGS_RELEASE="/INCREMENTAL:NO /DEBUG:NONE"
   -DCMAKE_SHARED_LINKER_FLAGS_RELEASE="/INCREMENTAL:NO /DEBUG:NONE"
   -DCMAKE_CUDA_FLAGS_RELEASE="-Xcompiler=/Z7"   # needed for sccache only; don't ship
   ```
   Then post-link, if PDBs were produced, delete them before `python -m build --wheel`:
   ```powershell
   Remove-Item build\*.pdb -Force -Recurse
   ```
2. **Do not bundle full CUDA runtime DLLs.** Only bundle:
   - `cudart64_12.dll` (~600 KB) — required for import
   - `cublas64_12.dll` (~115 MB) — required for matmul
   - `cublasLt64_12.dll` (~450 MB as of 12.4) — required for matmul
   This is unavoidable for convenience ("just pip install and go"), but it's the biggest wheel contributor. `cublasLt64_12.dll` alone is why the wheel is ~500 MB. You can drop cuBLAS if you use `GGML_CUDA_FORCE_MMQ=ON` (which the workflow already does, line 159) — this routes matmul through ggml's custom CUDA kernels instead of cuBLAS. **Verify after building** that `ggml-cuda.dll` does not link `cublas` anymore:
   ```powershell
   & dumpbin.exe /DEPENDENTS llama_cpp\lib\ggml-cuda.dll | Select-String "cublas"
   ```
   If this is empty, you can skip bundling cublas and cublasLt and drop ~550 MB. **This is the single biggest wheel-size win available.**
3. **Restrict arches to what the project actually targets.** 70 is V100/Tesla — drop unless you have a known user on it. Project target per user spec is "RTX 30xx/40xx", which is 86 and 89. Add 80 (A100) and 90 (H100/RTX 50xx Blackwell is 90/100) for safety. `70-real;75-real` alone saves ~50 MB and most users never touch them.
4. **Post-build size assertion:**
   ```powershell
   $wheel = Get-Item dist\*.whl
   if ($wheel.Length -gt 400MB) {
     Write-Warning "Wheel size $($wheel.Length / 1MB) MB exceeds target 400 MB"
     # optional: fail the build
   }
   ```

**Warning signs:**
- Wheel > 400 MB.
- `dumpbin /DEPENDENTS llama.dll` lists `cublas64_12.dll` despite `GGML_CUDA_FORCE_MMQ=ON`.
- `.pdb` files present next to `.pyd` in the wheel (extract and inspect).
- gh-pages repo size approaching 1 GB.

**Phase to address:** Phase 2 (build tuning) — immediately after the build first succeeds. Size is a real publish-gate constraint (gh-pages quota).

---

### Pitfall 11: Runtime DLL mismatch — wheel works on build host, fails on user box

**What goes wrong:**
Wheel installs fine on the user's machine, `from llama_cpp import Llama` throws `OSError: [WinError 126] The specified module could not be found`. The user has CUDA 12.6 driver installed (perfectly fine, forward-compatible). The wheel was built against CUDA 12.4.1 runtime and bundles `cudart64_12.dll` v12.4.1 — but that DLL has a dependency on `ucrtbase.dll` / `vcruntime140.dll` / `msvcp140.dll` from the MSVC runtime on the *build* machine. The user's Windows doesn't have the exact VC++ Redistributable installed.

**Why it happens:**
1. **MSVC runtime dependency.** All MSVC-compiled DLLs link `/MD` against the dynamic VC++ redistributable. If the user doesn't have the matching VC++ redist, DLL load fails. Ships with VS 2015/17/19/22 as `vcredist_x64.exe`. Almost every gaming/dev machine has it; many server/sandbox/container machines do not.
2. **CUDA forward compatibility, narrow window.** A wheel compiled against CUDA 12.4.1 *runtime* (cudart 12.4.1) will work on driver ≥ 525.60.13 (Linux) / ≥ 527.41 (Windows) according to [NVIDIA's CUDA Compatibility doc](https://docs.nvidia.com/deploy/cuda-compatibility/). Drivers older than that reject the 12.4 runtime with `cudaErrorInsufficientDriver`. RTX 40xx launched on driver 520.x; RTX 50xx requires 570.x+. The realistic user floor for "RTX 30xx/40xx" is driver 530+, safely above 12.4's requirement. **But**: the 12.4 *runtime* bundled in the wheel requires 527.41+, while some users still run 520-era drivers if their GPU is old.
3. **Driver major bump.** Wheel built against cudart 12.x works on driver for CUDA 12.x **or newer** (forward compat). Does NOT work on driver for CUDA 11.x. No backward compatibility.
4. **Duplicate DLLs on PATH.** User already has PyTorch installed, which bundles its *own* `cudart64_12.dll` (possibly a different patch version) in `site-packages/torch/lib`. Windows loader uses the first match — PyTorch's cudart wins if torch was imported first. Could be a different patch level; usually benign but occasionally crashes on cublas init.

**How to avoid:**
1. **Bundle the VC++ runtime DLLs in the wheel**:
   - `vcruntime140.dll`
   - `vcruntime140_1.dll`
   - `msvcp140.dll`
   They're ~100 KB each, copy from `$env:VCToolsRedistDir\x64\Microsoft.VC143.CRT\` at build time into the wheel's `llama_cpp/lib/` dir. Wheel becomes self-contained for MSVC runtime.
2. **Bundle cudart64_12.dll matching the build.** Already done implicitly if `CMAKE_CUDA_RUNTIME_LIBRARY=Shared` — just verify with `dumpbin`.
3. **Document minimum driver in the README:**
   - CUDA 12.4 wheel → Windows driver ≥ 551.61 (12.4 release driver) or any newer.
   - Forward compat works up to CUDA 12.x drivers; will NOT work on CUDA 11.x drivers.
4. **For CI, consider using `delvewheel`** (the Windows equivalent of `auditwheel repair` / `delocate`) to bundle *all* transitive DLL dependencies:
   ```yaml
   - run: pip install delvewheel
   - run: |
       delvewheel repair dist\*.whl -w dist\repaired --add-path "$env:CUDA_PATH\bin"
       Move-Item -Force dist\repaired\*.whl dist\
   ```
   `delvewheel` walks the DLL dependency tree and bundles everything (except Windows-provided DLLs) into the wheel. This is the canonical Windows-wheel-bundling tool as of 2025+. It will also rename bundled DLLs with a hash to avoid PATH collisions with PyTorch/other wheels.
5. **Smoke test the wheel from a fresh Python venv** on the same runner (not the build environment's Python), to catch "works on build host because it has CUDA installed, broken elsewhere" issues:
   ```powershell
   # Delete CUDA_PATH and all CUDA bin dirs from PATH before smoke test
   $env:PATH = ($env:PATH -split ';' | Where-Object { $_ -notlike "*CUDA*" -and $_ -notlike "*cuda*" }) -join ';'
   $env:CUDA_PATH = ''
   python -m venv smoke-venv
   smoke-venv\Scripts\activate
   pip install dist\*.whl
   python -c "import llama_cpp; llm = llama_cpp.Llama(...)"
   ```
   This is the *strongest* signal the CPU-only runner can give about end-user behavior.

**Warning signs:**
- Wheel works from the build directory but not from a fresh venv on the same machine.
- User reports `WinError 126` / `LoadLibrary failed`.
- `dumpbin /DEPENDENTS` of the shipped `.pyd` lists DLLs that are not bundled in the wheel (except for Windows API DLLs like `KERNEL32.dll`).

**Phase to address:** Phase 3 (smoke test hardening with `delvewheel` + clean-PATH venv test) + Phase 5 (user docs on minimum driver).

---

### Pitfall 12: llama.cpp submodule SHA drift — wheels built from "main" are not reproducible

**What goes wrong:**
Today's CI run produces `llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl` with llama.cpp at commit `227ed28e1`. Tomorrow's CI run, same trigger, same inputs, produces a wheel with the same filename but llama.cpp at `a1b2c3d4e` — because someone updated `.gitmodules` to track `main`, or someone ran `git submodule update --remote` in CI. Two users install "the same" wheel and get different behaviour.

**Why it happens:**
The repo **does** pin submodules by SHA (standard git submodule behavior) — good. BUT:
1. If CI ever runs `git submodule update --remote` (which explicitly advances to the tracked branch tip), pinning breaks.
2. If a developer updates the submodule locally and commits, the wheel version string (`llama_cpp_python-0.3.20`) doesn't change, but the binary does. Two wheels with the same filename but different contents is a cache-invalidation disaster for pip / gh-pages.
3. The version number `0.3.20` comes from `pyproject.toml`; llama.cpp bumps don't automatically bump it. Users have no way to distinguish "wheel 0.3.20 with llama.cpp X" from "wheel 0.3.20 with llama.cpp Y".

**How to avoid:**
1. **Embed the llama.cpp SHA in the wheel's version metadata.** Use a PEP 440 local version:
   ```
   0.3.20+cu124.llamacppSHA227ed28e
   ```
   Done via `pyproject.toml` dynamic version or by rewriting at build time:
   ```powershell
   $llamasha = (git -C vendor/llama.cpp rev-parse --short HEAD)
   $env:SETUPTOOLS_SCM_PRETEND_VERSION = "0.3.20+cu124.ll${llamasha}"
   ```
   This makes the wheel filename unique per submodule SHA and gives users something to report in bug tickets.
2. **Never use `--remote` in CI submodule init.** `actions/checkout@v4` with `submodules: recursive` does the right thing by default (checks out the pinned SHA). Leave it alone.
3. **Gate wheel publish on a clean `git status` of the submodule.** If the submodule was dirty at build time, refuse to publish:
   ```powershell
   $dirty = git -C vendor/llama.cpp status --porcelain
   if ($dirty) { throw "llama.cpp submodule has uncommitted changes — refusing to publish non-reproducible wheel" }
   ```
4. **Record the submodule SHA in the GitHub release notes** (or a `RELEASE_INFO.txt` next to the wheel on gh-pages). Lets users map "I have bug X with wheel 0.3.20" to a specific llama.cpp commit.

**Warning signs:**
- Two wheels with identical filenames but different SHA-256 hashes.
- Release notes don't mention llama.cpp version.
- `git submodule status` on fresh checkout shows `+` prefix (submodule at different SHA than recorded in superproject).

**Phase to address:** Phase 2 (build pipeline) — bake submodule SHA into the wheel version string from the start. Cheap and mechanical.

---

### Pitfall 13: Upstream workflow drift — fork diverges invisibly from abetlen's base

**What goes wrong:**
The user spec's plan is to create a *new* file `build-wheels-cuda-windows.yaml`, leaving the upstream `build-wheels-cuda.yaml` untouched. Good for merge hygiene. **But** the new Windows workflow depends on the same scripts / patterns / versions as the upstream Linux workflow. Six months later, upstream updates `conda-incubator/setup-miniconda` to `@v4` (breaking change in environment activation), bumps the Jimver action from `@master` to `@v0.2.23`, switches `actions/cache` to v5. Our Windows fork doesn't pick up these updates because it's a separate file. One day our workflow breaks with an inscrutable error, and we have no context for what upstream learned.

**Why it happens:**
1. Fork isolation is a double-edged sword: clean merges, but no inheritance of upstream fixes.
2. No one writes tests for CI workflows; "it works" is the only signal.
3. The upstream `build-wheels-cuda.yaml` is the *reference implementation* we copy-pasted from; when they fix a bug there, we don't know.
4. Dependabot on GitHub Actions tracks versions pinned in *each* workflow file independently — it will update the upstream file's pins but not our Windows-only file's pins unless both use the same version pins.

**How to avoid:**
1. **Subscribe to upstream activity on the original workflow file.** GitHub has file-level watch, sort of: set up a GitHub notification on pushes that touch `.github/workflows/build-wheels-cuda.yaml` via a custom script or third-party tool (e.g., a scheduled workflow that diffs upstream's file monthly and opens an issue if it changed). Simple version:
   ```yaml
   # .github/workflows/check-upstream-drift.yml
   on:
     schedule: [{cron: '0 7 * * 1'}]
   jobs:
     diff:
       runs-on: ubuntu-22.04
       steps:
         - uses: actions/checkout@v4
         - run: |
             curl -s https://raw.githubusercontent.com/abetlen/llama-cpp-python/main/.github/workflows/build-wheels-cuda.yaml > upstream.yaml
             diff .github/workflows/build-wheels-cuda.yaml upstream.yaml > drift.diff || true
             if [ -s drift.diff ]; then
               gh issue create --title "Upstream build-wheels-cuda.yaml drifted" --body "$(cat drift.diff)"
             fi
           env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
   ```
2. **Enable Dependabot for GitHub Actions**: add `.github/dependabot.yml` with `package-ecosystem: github-actions`. It keeps action versions in sync across all workflow files, reducing per-file drift.
3. **Annotate each inherited block** with a comment including the upstream commit SHA it was copied from:
   ```yaml
   # Inherited from abetlen/llama-cpp-python @ 1b1a320 (build-wheels-cuda.yaml L78-86)
   # 2026-04-15: adapted for windows-2022 / VS 2022 paths
   - name: Get Visual Studio Integration
     ...
   ```
   Makes future "should I rebase this chunk?" decisions tractable.
4. **Track [upstream #2136](https://github.com/abetlen/llama-cpp-python/issues/2136)** (Quansight funding offer). If abetlen responds or the project gets a new maintainer, Windows CUDA might return upstream and our fork becomes obsolete. Check quarterly.

**Warning signs:**
- Last update to the workflow file was > 6 months ago.
- Upstream `build-wheels-cuda.yaml` has had 5+ commits since we forked.
- Action versions in our workflow (`@v2`, `@v3`) are behind upstream (`@v4`).

**Phase to address:** Phase 6 (ongoing maintenance) — set up drift-check and Dependabot as soon as v1 ships.

---

## Moderate Pitfalls

### Pitfall 14: conda/mamba default shell activation breaks mid-job

**What goes wrong:**
Early steps activate `llamacpp` env; by step 6, `python` resolves to `C:\hostedtoolcache\windows\Python\3.11.*\x64\python.exe` (setup-python's install) not `$CONDA_PREFIX\python.exe` (mamba's install). `python -m build` runs against the wrong interpreter; wheel gets tagged wrong.

**Why it happens:**
The current workflow uses `defaults: run: shell: pwsh`, which does **not** source conda activation between steps. Each step starts a fresh pwsh session. `conda activate llamacpp` persisted via `conda init powershell` at runner provisioning time *might* pick up the env — or might not, depending on profile loading. `setup-miniconda` automatically adds `$CONDA_PREFIX\condabin` to PATH via GITHUB_PATH so *some* conda commands resolve, but `python` may still hit the setup-python one first.

**How to avoid:**
1. **Use `shell: bash -el {0}`** on steps that need the conda env activated — `-l` reads login profile which sources conda init.
2. **Or, per step, resolve python explicitly:**
   ```powershell
   $py = Join-Path $env:CONDA_PREFIX 'python.exe'
   & $py -m build --wheel
   ```
3. **Drop `setup-python` entirely** if mamba provides the Python — the spec's matrix has `pyver` driving both, so you're installing Python twice anyway. Pick one.
4. **Assert before building:**
   ```powershell
   where.exe python
   python --version
   # Assert the first hit is under CONDA_PREFIX
   ```

**Phase to address:** Phase 1.

---

### Pitfall 15: Windows long path limit — CUDA build paths exceed 260 characters

**What goes wrong:**
Build starts, runs for 10 minutes, then `fatal error C1083: Cannot open compiler generated file: 'D:\a\llama-cpp-python\llama-cpp-python\build\cp311-win_amd64\vendor\llama.cpp\ggml\src\ggml-cuda\CMakeFiles\ggml-cuda.dir\template-instances\mmq-instance-f16-tile32.cu.obj': No such file or directory`. The path is 289 characters; MSVC refuses. Windows-specific, nobody hits it on Linux.

**Why it happens:**
Windows default path limit is 260 chars (`MAX_PATH`). Python wheel builds tend to nest deeply: `{runner}/{repo}/build/{wheel tag}/{vendor submodule}/{cmake output}/{object dir}/{template instance}.cu.obj`. llama.cpp's ggml-cuda has deeply-nested template-instance files for every matmul/quant combination. Add sccache's temp dirs and you're well over 260.

**How to avoid:**
1. **Enable long paths on the runner early:**
   ```powershell
   # At the very first step of the job
   Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' `
     -Name 'LongPathsEnabled' -Value 1
   git config --system core.longpaths true
   ```
   (windows-2022 may already have `LongPathsEnabled=1` by default — verify.)
2. **Keep the build directory short:** `$env:CMAKE_BUILD_PARALLEL_LEVEL = 4; cmake -B C:\b -S .` using a short absolute path for `-B`.
3. **If still hitting it:** enable `/longPathAware` on the MSVC linker, or re-root the build: `cd /; mklink /J C:\b D:\a\long\path\build`.

**Phase to address:** Phase 1 (workflow scaffold).

---

### Pitfall 16: GitHub Actions 6h timeout hit by cold sccache + cold mamba + cold Jimver download

**What goes wrong:**
First run after a cache eviction takes 4h55m. Second run that actually needs to debug a failure has 5 minutes left. Iteration is hell.

**Why it happens:**
- Cold CUDA Jimver download: ~5 min (3 GB).
- Cold mamba install of `cuda-toolkit`: 10–15 min.
- Cold llama.cpp + llama-cpp-python compile (no sccache): 30–45 min just for CUDA kernels, another 5–10 for C++.
- Cold submodule checkout: 2 min.
- Smoke test: 2 min.
- Total cold: ~50–70 min with everything working. Add debugging re-runs for VS/nvcc mismatch: could approach 2h before you give up on a bad pin.

6h is not realistically hit on a good build, but it **is** hit if a build hangs (e.g., nvcc enters some infinite template resolution loop on a bad input). The "cold path" is the dangerous one.

**How to avoid:**
1. **Save caches on failure** — the user spec already calls this out. Use `hendrikmuhs/ccache-action@v1.2` which saves on post-step regardless of job outcome, or explicitly use `actions/cache/save@v4` as a separate step at the end, so cache survives build failures. For the other caches (CUDA installer, mamba pkgs), use `actions/cache/save@v4` with `if: always()`:
   ```yaml
   - name: Save CUDA installer cache
     uses: actions/cache/save@v4
     if: always() && steps.cuda-installer-cache.outputs.cache-hit != 'true'
     with:
       path: cudainstaller.zip
       key: cuda-installer-${{ matrix.cuda }}-windows
   ```
2. **Set a per-step timeout** so a hung step doesn't eat the whole budget:
   ```yaml
   - name: Build Wheel
     timeout-minutes: 90
     run: ...
   ```
3. **Parallelize:** `-G Ninja` with `CMAKE_BUILD_PARALLEL_LEVEL=4` (windows-2022 runner has 4 cores). Default is 1 for MSBuild generator; 4x speedup just from generator switch.

**Phase to address:** Phase 2.

---

## Minor Pitfalls

### Pitfall 17: PEP 503 name normalization trap in the pip index

**What goes wrong:**
`index.html` links use `llama_cpp_python-0.3.20-...whl`, but pip requests the index at `.../llama-cpp-python/` (hyphen, not underscore). If the directory on gh-pages is `.../llama_cpp_python/`, pip can't find it.

**Why it happens:**
PEP 503 says: normalize the project name to `re.sub(r"[-_.]+", "-", name).lower()` → `llama-cpp-python`. Wheel **filenames** keep the distribution name as declared in `pyproject.toml` (usually with underscores in `_` form per wheel spec), but **index directory names** must be the normalized form.

**How to avoid:**
- Index path: `whl/cu124/llama-cpp-python/index.html` (hyphen, always).
- Wheel filenames inside: `llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl` (underscore, always — this is what `bdist_wheel` emits).
- Link in `index.html`: `<a href="llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl">...</a>` (underscore — matches actual filename).
- Install command: `pip install llama-cpp-python` or `pip install llama_cpp_python` — both work (pip normalizes both to request `llama-cpp-python/`).

**Warning signs:**
- `pip install llama-cpp-python --extra-index-url=...` says `No matching distribution` but the file is clearly on gh-pages.
- `curl <index_url>` returns 404 at the directory level.

**Phase to address:** Phase 4 (publishing) — use a tested index-generation script, not hand-rolled HTML.

---

### Pitfall 18: `actions-gh-pages@v4` requires `contents: write`, which the default token has on the current workflow — but only on pushes, not PRs

**What goes wrong:**
Workflow works on push/manual dispatch, fails on PRs from forks with `Permission denied`.

**Why it happens:**
`GITHUB_TOKEN` on PRs from forks is **read-only** for security. Can't push to gh-pages from a fork PR.

**How to avoid:**
- Workflow already only triggers on `workflow_dispatch` per spec — this pitfall doesn't apply in v1.
- When adding tag/release triggers in v2, ensure the gh-pages publish only runs on pushes to trusted branches, not on PR triggers.

**Phase to address:** Phase 5 (v2 triggers, if ever).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Use `--allow-unsupported-compiler` to unblock build | Build stops failing | **Wheel segfaults at runtime.** Users report crashes weeks later. Entire project credibility tanks. | **Never.** Explicit ban in user spec. |
| Pin `windows-2022` without pinning VS toolset | One less step in the YAML | Build silently breaks on next image rotation | Never — the whole project exists because of this failure mode |
| Hardcode VS 2019 Enterprise path | Faster YAML write | Silent no-op on `windows-2022`; confusing when debugged | Never — use dynamic detection |
| Skip the smoke test ("the build succeeded, that's enough") | Shaves 2 minutes off CI | Ships segfaulting wheels. Publish gate defeated. | Never |
| Publish with `force_orphan: true` or full tree overwrite | Clean history, one wheel visible | Wipes every other wheel from gh-pages on every run | Never — breaks the whole multi-version publish model |
| Skip `delvewheel` / DLL bundling | Smaller wheel by ~500 MB | Works on build host, breaks on user machines without cuBLAS installed | MVP if `GGML_CUDA_FORCE_MMQ=ON` and you drop cublas entirely (then you only need to bundle cudart) |
| Skip llama.cpp SHA embed in wheel version | Simpler version string | Bug reports become untraceable across submodule bumps | Never — it's a 3-line script |
| Copy-paste from upstream workflow without annotation | Fast initial port | Unmaintainable drift; no way to rebase | Only if drift-check workflow is also added |
| Use `shell: pwsh` everywhere | Uniform YAML | Conda activation breaks; double Python interpreters | Never for steps that use conda — use `bash -el {0}` |
| Run smoke test from build directory (not fresh venv) | Simpler YAML | Passes on CI, fails for users whose PATH lacks CUDA | MVP if `delvewheel` is in use |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `microsoft/setup-msbuild@v2` | `vs-version: '[17.0,18.0)'` (too wide — matches anything) | `vs-version: '[17.9,17.10)'` (CUDA 12.4.1 whitelist) |
| `Jimver/cuda-toolkit@master` | Pin to `@master` — breaks when Jimver changes API | Pin to `@v0.2.23` or latest tagged release |
| `conda-incubator/setup-miniconda@v3` | Use `shell: pwsh` everywhere, expect conda activation | Use `shell: bash -el {0}` for conda-dependent steps |
| `actions/cache@v4` | Omit `restore-keys` — no fallback on exact key miss | Always supply `restore-keys:` hierarchy for partial matches |
| `peaceiris/actions-gh-pages@v4` | `force_orphan: true` — "clean history is nice" | `keep_files: true`, `destination_dir: whl/cuXYZ/llama-cpp-python` |
| `sccache` | `CMAKE_C_COMPILER_LAUNCHER=sccache` without `/Z7` flag fixup | Pair with `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded` and `CMP0141=NEW` |
| `softprops/action-gh-release@v2` | Use for gh-pages publish — wrong tool | Use for Release assets only; `peaceiris/actions-gh-pages` for pip index |
| `actions/checkout@v4` | Omit `submodules: recursive` — llama.cpp not checked out | Always `submodules: recursive`; never `--remote` |
| `delvewheel` | Not used — wheel depends on host PATH | Always repair wheel before publish; test in clean-PATH venv |
| `hendrikmuhs/ccache-action@v1.2` | Use `variant: ccache` on Windows | Use `variant: sccache` — ccache has no MSVC support |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| sccache cache miss due to `/Zi` | Build time 80%+ of cold on "warm" rerun | Force `/Z7` via CMP0141 | Every rebuild after first |
| Over-broad CUDA arches (sm_70..sm_90 all real+virtual) | Wheel > 600 MB, link step > 5 min | Limit to `70-real;75-real;80-real;86-real;89-real;90-real;90-virtual` (user spec already correct) | Wheel size limit (gh-pages 1 GB) |
| Ninja generator not used | Build takes 45+ min (MSBuild is single-threaded-ish) | `-G "Ninja"` or `-G "Ninja Multi-Config"` + `CMAKE_BUILD_PARALLEL_LEVEL=4` | Every cold build |
| CDN cache not invalidated | `pip install` sees stale wheel list for 10–30 min after push | Post-publish HTTP probe loop with 60s sleeps | Every publish (always-on, minor) |
| Mamba package cache not persisted | `mamba install cuda-toolkit` runs 10 min every build | `actions/cache` on `~/conda_pkgs_dir` | Every cold build after cache eviction (7-day default for GitHub cache) |
| CUDA installer not cached | `Invoke-RestMethod` 3 GB zip every run (5 min) | `actions/cache` on `cudainstaller.zip` keyed by CUDA version | Every run without cache hit |

---

## Security Considerations

Domain-specific, not OWASP-level.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Publish wheels without SHA-256 digests in index.html | User can't verify wheel integrity; MITM replace | Include `#sha256=<hash>` fragment in each `<a href>` in the index (PEP 503 supports this) |
| Accept `workflow_dispatch` inputs with shell-interpolated Python code | Anyone with repo write can run arbitrary code on the runner | Whitelist inputs: `python_version: enum [3.9, 3.10, 3.11, 3.12, 3.13]`; `cuda_version: enum [12.1.1, 12.2.2, 12.3.2, 12.4.1]` |
| PAT with `repo` scope hardcoded in workflow | Leaked PAT gives full repo write | Use `GITHUB_TOKEN` with minimum `permissions: { contents: write }` scoped to the publish job |
| Trust Jimver's master branch | Supply-chain: Jimver could push a malicious action update | Pin Jimver to a specific SHA, not `@master` / `@v0.2` |
| Download CUDA installer from a raw URL not signed by NVIDIA | Potential download tampering | Verify SHA-256 of `cudainstaller.zip` against a known-good hash after download |
| gh-pages branch accepts writes from any action | Any workflow in the repo could vandalize the pip index | Use a dedicated `publish-wheels-windows-cuda.yml` workflow with narrow permissions; don't give `contents: write` to unrelated workflows |

---

## "Looks Done But Isn't" Checklist

- [ ] **Build succeeds** → did it set `--allow-unsupported-compiler`? Grep YAML and CMake output for it.
- [ ] **Wheel installs** → does `python -c "import llama_cpp"` work in a *fresh venv on a machine without CUDA installed*?
- [ ] **Wheel loads** → does `Llama(model_path=..., n_gpu_layers=0)` generate at least 1 token without segfault?
- [ ] **Smoke test passes** → did the smoke test exercise Python bindings, or just import the module?
- [ ] **Wheel tag correct** → does the filename match `cp<pyver>-cp<pyver>-win_amd64`, not `abi3` or `none-any`?
- [ ] **Wheel size reasonable** → is it < 400 MB? If > 500 MB, `cublasLt64_12.dll` is probably bundled unnecessarily.
- [ ] **CUDA arches baked in** → does `cuobjdump --list-elf ggml-cuda.dll` show `sm_89` for RTX 40xx users?
- [ ] **Index published** → did you `curl` the gh-pages URL after waiting 15 min, and does the new wheel show up?
- [ ] **Submodule pinned** → does the wheel version string include llama.cpp SHA?
- [ ] **VS pinned** → does the YAML specify `vs-version: '[17.x,17.y)'` with a narrow range, not a wide one?
- [ ] **Not publishing destructively** → after publishing a 3.12 wheel, is the existing 3.11 wheel still in the index?
- [ ] **Concurrency group set** → does the workflow have `concurrency: group: gh-pages-publish`?
- [ ] **Runner image watchdog** → does the canary / weekly scheduled run exist?
- [ ] **Upstream drift check** → is there a workflow that checks for changes to upstream's `build-wheels-cuda.yaml`?
- [ ] **DLL bundling verified** → does `dumpbin /DEPENDENTS` on the shipped `.pyd` show only bundled + Windows API DLLs?

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Published a segfaulting wheel to gh-pages | **HIGH** | 1) Immediately push a new commit to gh-pages that removes the bad wheel from `whl/cuXYZ/llama-cpp-python/` AND deletes its entry from `index.html`. 2) Post a GitHub issue tagging the bad wheel SHA-256. 3) Warn users on README. 4) Pip doesn't recognize "yanked" wheels from a custom index; removal is the only way. |
| gh-pages wheel wiped by concurrent publish race | MEDIUM | 1) Re-run the workflow for the wiped (Python × CUDA) combo. 2) Add `concurrency:` block before next run. 3) Consider `git revert` on gh-pages if history is clean enough. |
| VS image rotation broke CI overnight | MEDIUM | 1) Pin `vs-version` tighter. 2) If no VS in range on `windows-2022`, switch to `windows-2025` temporarily. 3) Downgrade CUDA one minor version if nothing works (12.3 was stable against wider VS range). 4) Canary workflow should have caught this — add one if absent. |
| CUDA installer cache corrupted | LOW | 1) Bump cache key version (`cuda-installer-v2-${{ matrix.cuda }}-windows`). 2) Re-run. |
| sccache cache poisoned / no hits | LOW | 1) Bump sccache key name. 2) `sccache --zero-stats` in a fresh run. 3) Re-verify /Z7 is forced. |
| Wheel too big for gh-pages quota | MEDIUM | 1) Move large wheels to GitHub Releases, link from gh-pages index. 2) Shrink wheel (drop cublas bundling via FORCE_MMQ). 3) Clean up old wheel versions from gh-pages. |
| Fork diverged from upstream, can't rebase | HIGH | 1) Never rebase — keep fork-specific workflow in a separate file. 2) If caught up in merge hell, cherry-pick upstream fixes selectively. 3) Drift-check workflow prevents this. |

---

## Pitfall-to-Phase Mapping

Presumes a rough 6-phase structure (scaffold → build → cache → smoke test → publish → maintenance). Adjust as roadmap evolves.

| # | Pitfall | Prevention Phase | Verification |
|---|---------|------------------|--------------|
| 1 | `-allow-unsupported-compiler` segfault | **Phase 1** (scaffold) | Grep YAML: `grep -r 'allow-unsupported' .github/` must return empty. Smoke test (Phase 3) catches regressions. |
| 2 | VS/nvcc image rotation | **Phase 1** (scaffold) + **Phase 6** (canary) | `setup-msbuild` has narrow `vs-version`; preflight step asserts `cl.exe` version ceiling; weekly canary green. |
| 3 | `CUDA_PATH` double-install confusion | **Phase 1** (scaffold) | `nvcc --version` output matches requested CUDA version; single `CUDA_PATH` in env at build time. |
| 4 | MSBuildExtensions copy to non-existent VS 2019 path | **Phase 1** (scaffold) | Dynamic path detection with strict-mode error on empty match. |
| 5 | sccache `/Zi` cache miss | **Phase 2** (caching) | `sccache --show-stats` hit rate > 50% on rebuild; CMP0141 + Embedded set. |
| 6 | Wheel mis-tagged (abi3) | **Phase 2** (build) + **Phase 3** (smoke test) | Regex check on wheel filename before publish step. |
| 7 | CPU-only smoke-test false negative | **Phase 3** (smoke test) | Smoke test augmented with `cuobjdump --list-elf` arch verification; doc explicitly says "not a GPU test". |
| 8 | gh-pages concurrent publish race | **Phase 4** (publishing) | `concurrency: group: gh-pages-publish`; `keep_files: true`; post-publish index assertion lists *all* wheels. |
| 9 | Fastly CDN stale after push | **Phase 4** (publishing) + **Phase 5** (docs) | Post-publish verification polls index URL for 15 min; README mentions the delay. |
| 10 | Wheel bloat > 500 MB | **Phase 2** (build) | Wheel size assertion (< 400 MB target); `dumpbin /DEPENDENTS` check for unwanted cublas. |
| 11 | Runtime DLL mismatch on user machine | **Phase 3** (smoke test) + **Phase 5** (docs) | `delvewheel repair`; clean-PATH venv smoke test; README states minimum driver. |
| 12 | Submodule SHA drift | **Phase 2** (build) | Version string includes `+ll<sha>`; clean-submodule check before publish. |
| 13 | Upstream workflow drift | **Phase 6** (maintenance) | Scheduled drift-check workflow opens issue on diff; Dependabot configured. |
| 14 | conda/pwsh activation PATH confusion | **Phase 1** (scaffold) | `where python` assertion before `python -m build`. |
| 15 | Windows long paths | **Phase 1** (scaffold) | `LongPathsEnabled` registry set; `core.longpaths=true`. |
| 16 | 6h runner timeout | **Phase 2** (caching) | Per-step `timeout-minutes`; `always()` cache save. |
| 17 | PEP 503 name normalization | **Phase 4** (publishing) | Index generator normalizes directory name; install documented both ways works. |
| 18 | PR from fork can't publish | **Phase 5** (v2 triggers) | Publish step guarded with `if: github.event_name != 'pull_request'`. |

---

## Cross-References to Upstream Context

| Upstream issue / commit | Our pitfall | Relevance |
|-------------------------|-------------|-----------|
| [#1543](https://github.com/abetlen/llama-cpp-python/issues/1543) | **Pitfall 1** (segfaulting wheels) | Direct precedent: reporter confirmed `-allow-unsupported-compiler` crashes at runtime |
| [#1551](https://github.com/abetlen/llama-cpp-python/issues/1551) | **Pitfall 2** (VS/nvcc rotation) | VS update broke build |
| [#1838](https://github.com/abetlen/llama-cpp-python/issues/1838) | **Pitfall 2** | Same failure mode, different image version |
| [#1894](https://github.com/abetlen/llama-cpp-python/issues/1894) | **Pitfall 2** | Same failure mode |
| [#2068](https://github.com/abetlen/llama-cpp-python/issues/2068) | Context for whole project | User demanding cu128 wheels; maintainer silent |
| [#2127](https://github.com/abetlen/llama-cpp-python/issues/2127) | **Pitfall 17** + gen. | `whl/cu125` 404 — index maintenance gap |
| [#2136](https://github.com/abetlen/llama-cpp-python/issues/2136) | **Pitfall 13** (drift) | Quansight funding offer unanswered — indicates upstream is paused; fork is long-term |
| [commit 98fda8c](https://github.com/abetlen/llama-cpp-python/commit/98fda8c) | **Pitfall 2** | The commit that disabled Windows CUDA in July 2025 |
| [commit 4f17ae5](https://github.com/abetlen/llama-cpp-python/commit/4f17ae5) | **Pitfall 1/2** | Removed CUDA 12.5/12.6 due to VS incompatibility |
| [cupy/cupy#8578](https://github.com/cupy/cupy/issues/8578) | **Pitfall 2** | Sibling project hitting same VS/nvcc rotation |
| [nerfstudio-project/nerfstudio#3171](https://github.com/nerfstudio-project/nerfstudio/issues/3171) | **Pitfall 1** | Documents exact nvcc error message and `--allow-unsupported-compiler` workaround |
| [actions/runner-images#10448](https://github.com/actions/runner-images/issues/10448), [#10575](https://github.com/actions/runner-images/issues/10575), [#10954](https://github.com/actions/runner-images/issues/10954) | **Pitfall 2** | VS 17.11/17.12 rollout history |
| [ccache/ccache#1040](https://github.com/ccache/ccache/issues/1040), [Mozilla bz 1000726](https://bugzilla.mozilla.org/show_bug.cgi?id=1000726) | **Pitfall 5** | Historical evidence of `/Zi` not being cacheable in ccache/sccache |

---

## Sources

- NVIDIA: [CUDA 12.4 Installation Guide for Windows](https://docs.nvidia.com/cuda/archive/12.4.0/cuda-installation-guide-microsoft-windows/) — MSVC 193x supported range
- NVIDIA: [CUDA Compatibility / Forward Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/forward-compatibility.html) — driver ↔ runtime version matrix
- NVIDIA Developer Forums: [_MSC_VER 1940 and llama.cpp build](https://forums.developer.nvidia.com/t/msc-ver-is-1940-with-latest-vs-2022-upate-and-its-not-letting-me-build-llama-cpp/295748) — direct VS 17.10 / nvcc incident report
- [Quasar CUDA/MSVC compatibility matrix](https://quasar.ugent.be/files/doc/cuda-msvc-compatibility.html) — community-maintained VS/CUDA table
- [Mozilla sccache repo](https://github.com/mozilla/sccache) and [sccache /Zi issue](https://github.com/ccache/ccache/issues/1040) — PDB caching limitations
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) — `keep_files`, concurrency patterns
- [PEP 503](https://peps.python.org/pep-0503/) — name normalization rules
- [pypa name normalization spec](https://packaging.python.org/en/latest/specifications/name-normalization/)
- [GitHub Actions Concurrency docs](https://docs.github.com/en/actions/using-jobs/using-concurrency)
- [actions/runner-images](https://github.com/actions/runner-images) — windows-2022 VS versioning history
- User spec: `C:\claude_checkouts\llama-cpp-python\.planning\user_specs\001_WINDOWS_CUDA_WHEELS_CI_PLAN.md` § 7, 8, 9
- Existing workflow: `C:\claude_checkouts\llama-cpp-python\.github\workflows\build-wheels-cuda.yaml` lines 46–181

---
*Pitfalls research for: Windows CUDA Python wheel build CI with gh-pages pip index publishing*
*Researched: 2026-04-15*
