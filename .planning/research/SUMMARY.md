# Project Research Summary

**Project:** Windows CUDA Wheels CI for llama-cpp-python (fork)
**Domain:** GitHub Actions CI pipeline — Windows x64 CUDA Python wheel build + PEP 503 pip index on GitHub Pages
**Researched:** 2026-04-15
**Confidence:** HIGH

## Executive Summary

This is a **fork CI for a PyPI-shaped package** that needs to resurrect what upstream `abetlen/llama-cpp-python` disabled in July 2025 (commit `98fda8c`): Windows x64 CUDA wheels that actually work at runtime. Four researchers — stack, features, architecture, pitfalls — converge almost unanimously on the same handful of decisions. The project is not architecturally novel (the upstream workflow is ~80% reusable); the hard part is **one specific compatibility trap** that has broken Windows CUDA builds in this ecosystem for two years: the `nvcc` toolchain's strict `_MSC_VER` allow-list versus GitHub's moving `windows-2022` image, which now ships MSVC 14.50 (`_MSC_VER = 1950`) — outside CUDA 12.4.1's `[1910, 1950)` window. The historical "fix" of `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` silences the error and produces wheels that segfault on first `Llama(...)` call (upstream issue #1543). That flag is banned.

The recommended approach is **pin MSVC toolset 14.39 (VS 17.9) as a side-by-side install on the runner, activate it via `ilammy/msvc-dev-cmd@v1 toolset: 14.39`, and gate every publish on a smoke test in a fresh-venv job that proves the wheel loads a 27 KB `tiny-llama.gguf` and generates 2 tokens without crashing**. Architecturally, a **3-job DAG (build → smoke-test → publish)** with explicit artifact hand-off replaces upstream's monolithic job with `if:` branches. Publish runs on `ubuntu-latest` using `peaceiris/actions-gh-pages@v4` with `keep_files: true` so concurrent runs accumulate wheels across `whl/cuXXX/llama-cpp-python/` subdirectories instead of overwriting each other. Wheels are stored as GitHub Release assets (abetlen's proven pattern) while gh-pages holds only the PEP 503 HTML index.

The risks are real but bounded. **Image drift** (`windows-2022` rolling forward a new VS 17.x) can silently break the build overnight — mitigated by pinning the toolset (not just the VS version) and a weekly canary run. **Cache poisoning** from `/Zi` debug info caps sccache hit rate at ~0% unless `/Z7` is forced via `CMP0141=NEW` + `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded`. **Cache-on-failure is mandatory** — most dev iterations will be failing builds, and paying a 3 GB CUDA download each retry makes the loop unworkable. **CPU-only smoke testing** catches 80% of the historical failure modes but cannot detect broken device code — compensated with a `cuobjdump --list-elf` arch-verification step.

## Key Findings

### Recommended Stack

The stack is boring-by-design: every choice matches what ExLlamaV2, flash-attention-prebuild-wheels, and PyTorch's Windows CUDA wheel CI already do. The one non-standard element is the **explicit MSVC toolset downgrade**, which upstream's workflow does not currently do — and which is the root cause of its current breakage.

**Core technologies:**

- **Runner:** `windows-2022` (pinned, not `windows-latest`) — VS 2022 Enterprise 17.14 ships on the image; we downgrade the *active toolset* via `vcvarsall.bat`
- **MSVC toolset pin:** `14.39` (`_MSC_VER = 1939`, VS 17.9) installed via `vs_installer.exe modify --add Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64`, activated via `ilammy/msvc-dev-cmd@v1 toolset: 14.39` — the only reliable way to satisfy nvcc 12.4.1's `_MSC_VER < 1950` ceiling
- **CUDA toolkit:** 12.4.1 installed via `conda-incubator/setup-miniconda@v3` + `mamba install cuda-toolkit` (not Jimver's full installer — issue #382 documents it hangs on `windows-2022` for 12.5+); `Jimver/cuda-toolkit@v0.2.24` used only for the MSBuildExtensions link table
- **Build backend:** `python -m build` via scikit-build-core (NOT cibuildwheel — no Windows CUDA story, no GPU smoke-test hook)
- **Compiler cache:** `hendrikmuhs/ccache-action@v1.2` with `variant: sccache` (saves on failure in `post` step) + `CMAKE_{C,CXX,CUDA}_COMPILER_LAUNCHER=sccache`
- **Data caches:** `actions/cache@v4` with split restore/save pattern and `if: always()` save-on-failure
- **Publish:** `peaceiris/actions-gh-pages@v4` with `keep_files: true` and `destination_dir: whl/cu124/llama-cpp-python/`; wheels themselves stored as GitHub Release assets via `softprops/action-gh-release@v2`
- **Smoke runner:** `windows-2022` (fresh job, fresh venv, sparse checkout that omits `llama_cpp/` source so `import` resolves to `site-packages`)
- **Publish runner:** `ubuntu-latest` (5× cheaper per minute)

Full detail in STACK.md.

### Expected Features

Scoped explicitly to a **fork CI serving one downstream team** (APHP). Anti-feature list is as load-bearing as the feature list: PyPI publishing, GPG signing, full Python × CUDA matrix on every dispatch, delvewheel DLL bloat, auto-trigger on upstream tag — all rejected for v1.

**Must have (table stakes):**

- PEP 503-compliant HTML index at `whl/cu124/llama-cpp-python/index.html` with SHA-256 hash fragments and `data-requires-python` attributes
- Variant-keyed URL path (`/whl/cu124/`) matching abetlen's UX exactly
- Runtime smoke test gates publish — install wheel into fresh venv, load `tiny-llama.gguf`, generate 2 tokens
- Three-tier cache with save-on-failure (sccache + CUDA installer zip + mamba pkgs)
- `workflow_dispatch` only with `python_version` and `cuda_version` inputs (defaults 3.11, 12.4.1)
- Multiple versions coexist on index (append-only, `keep_files: true`)
- Ban on `-allow-unsupported-compiler` documented inline in YAML
- README install blurb: `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu124`

**Should have (v1.x, after single-combo stability):**

- `data-requires-python` attributes, build badge, changelog
- Multi-Python matrix (3.10/3.11/3.12) — only after single-combo green for ~1 month
- Multi-CUDA matrix (12.4/12.6) — bounded by VS compatibility wall
- `cuobjdump --list-elf` arch verification step

**Defer (v2+):**

- Sigstore / PEP 740 attestations, SBOM generation, auto-trigger on upstream tag, PEP 658 metadata exposure, delvewheel DLL vendoring, Python 3.13 / free-threaded

Full detail in FEATURES.md.

### Architecture Approach

Three jobs, linear `needs` chain, artifact-as-contract. The load-bearing boundary is **smoke-test ≠ build** — a same-job test imports from the `./llama_cpp/` source tree via `sys.path`, so wheel self-sufficiency is never exercised; splitting jobs with sparse checkout forces `pip install` + `import` to resolve against `site-packages`.

**Major components:**

1. **`build` (windows-2022)** — CUDA toolkit install, MSVC 14.39 toolset install + activation, cache restore, `python -m build --wheel`, artifact upload. Version derived from `llama_cpp/__init__.py` — same source scikit-build-core reads, so no drift.
2. **`smoke-test` (windows-2022, fresh runner, sparse checkout)** — download artifact, `pip install` in clean venv, `from llama_cpp import Llama; Llama('tiny-llama.gguf', n_ctx=64)('hi', max_tokens=2)`. Hard publish gate.
3. **`publish` (ubuntu-latest)** — checkout gh-pages, place wheel at `whl/cu<tag>/llama-cpp-python/<file>`, regenerate `index.html` from directory listing, `peaceiris/actions-gh-pages@v4 keep_files: true`. Release assets created in parallel.

**Cache key composition:**

```
sccache-${runner.os}-py${python_version}-cu${cuda_version}-submod${hashFiles('vendor/llama.cpp/.git/HEAD', '.gitmodules')}-cmake${hashFiles('CMakeLists.txt', 'pyproject.toml')}
```

All five inputs must appear in the primary key; `restore-keys` provides progressively shorter fallbacks.

Full detail in ARCHITECTURE.md.

### Critical Pitfalls

1. **`-allow-unsupported-compiler` produces segfaulting wheels** (upstream #1543) — silences nvcc's `_MSC_VER` check without fixing the ABI mismatch. **Mitigation:** Pin MSVC 14.39 via `ilammy/msvc-dev-cmd`; never add the flag; grep-assert absence in CI.
2. **`windows-2022` VS image rolls forward** — current image ships MSVC 14.50, outside CUDA 12.4.1's window. **Mitigation:** Side-by-side-install 14.39 toolset component; preflight step asserts `cl.exe /Bv` is under the ceiling; weekly canary run.
3. **CUDA_PATH double-install confusion** — upstream installs CUDA twice (Jimver zip + mamba) causing ABI mismatches. **Mitigation:** Pick one install path (mamba), unset all `CUDA_PATH_V*_*` variants, assert `nvcc --version` matches request.
4. **sccache cache miss with `/Zi` MSVC debug info** — PDBs shared across TUs can't be cached safely. **Mitigation:** Force `/Z7` via `CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=Embedded` + `CMAKE_POLICY_DEFAULT_CMP0141=NEW`; verify hit rate > 30%.
5. **gh-pages concurrent publish race** — simultaneous dispatches overwrite each other's wheels. **Mitigation:** `concurrency: group: gh-pages-publish, cancel-in-progress: false` + `keep_files: true` + regenerate `index.html` from actual directory listing.
6. **CPU-only smoke test false confidence** — runners have no GPU, so broken device code slips through. **Mitigation:** Augment with `cuobjdump --list-elf ggml-cuda.dll` arch-presence assertion; document "CI green ≠ GPU-verified".
7. **Fastly CDN stale for up to 15 min after gh-pages push** — `pip install` can't find new wheel despite successful push. **Mitigation:** Post-publish HTTP probe loop (15× 60s); document the delay in README.

Full detail (13 critical + 2 moderate + 2 minor) in PITFALLS.md.

## Implications for Roadmap

Dependencies are strict: every phase is a hard prerequisite for the next, and each phase closes a specific class of pitfall.

### Phase 1: Workflow scaffold + MSVC/CUDA pinning

**Rationale:** This is where the upstream failure lives. Until MSVC 14.39 is installed and activated, nvcc rejects the build; every other investment is wasted. Must be first.
**Delivers:** `.github/workflows/build-wheels-cuda-windows.yaml` with `workflow_dispatch` trigger, Python/CUDA dispatch inputs, checkout with submodules, MSVC 14.39 side-by-side install, `ilammy/msvc-dev-cmd` activation, single-path CUDA install via mamba, dynamic VS 2022 BuildCustomizations detection, `LongPathsEnabled` registry set.
**Addresses features:** `workflow_dispatch`-only, dispatch inputs, new workflow file separate from upstream.
**Avoids pitfalls:** #1 (allow-unsupported-compiler segfault), #2 (VS image rotation), #3 (CUDA_PATH double-install), #4 (MSBuildExtensions hardcoded path), #14 (pwsh/conda activation), #15 (Windows long paths).

### Phase 2: Build + caching + size discipline

**Rationale:** Build has to succeed before smoke test can test it; caches have to work on failure before the dev loop is usable; wheel size matters before publish (gh-pages quotas).
**Delivers:** `python -m build --wheel` producing `cp<ver>-cp<ver>-win_amd64`-tagged wheel under 400 MB; sccache with `/Z7` enforced via CMP0141; three-tier cache with `if: always()` save; `GGML_CUDA_FORCE_MMQ=ON` + `dumpbin` assertion that `cublas64_12.dll` is not linked; wheel version embeds llama.cpp short SHA (`0.3.20+cu124.ll<sha>`); per-step `timeout-minutes`; Ninja generator + `CMAKE_BUILD_PARALLEL_LEVEL=4`.
**Uses stack:** scikit-build-core, sccache, CMake Ninja, `CMAKE_CUDA_ARCHITECTURES=70-real;75-real;80-real;86-real;89-real;90-real;90-virtual`.
**Implements architecture:** `build` job, artifact upload, version derivation from `llama_cpp/__init__.py`.
**Avoids pitfalls:** #5 (sccache /Zi miss), #6 (wheel tag abi3/none-any), #10 (wheel bloat), #12 (submodule SHA drift), #16 (6h runner timeout).

### Phase 3: Smoke test + publish gate

**Rationale:** A wheel that compiles is not a wheel that works — proving otherwise is what makes this fork valuable vs upstream.
**Delivers:** `smoke-test` job on fresh windows-2022 runner with sparse checkout (omits `llama_cpp/` source); `pip install` of downloaded artifact in clean venv; `from llama_cpp import Llama; Llama('llm_for_ci_test/tiny-llama.gguf', n_ctx=64)('hi', max_tokens=2)` asserting no segfault; `cuobjdump --list-elf` arch-presence assertion; clean-PATH venv test; `llm_for_ci_test/tiny-llama.gguf` (27 KB) committed; wheel-tag regex assertion before smoke test.
**Implements architecture:** `smoke-test` job with `needs: build`, artifact download, sparse checkout.
**Avoids pitfalls:** #1 regression-catch, #6 tag-bug late detection, #7 (CPU-only false negative — augmented with cuobjdump), #11 (runtime DLL mismatch on user box).

### Phase 4: Publish (gh-pages index + release assets)

**Rationale:** Only runs once build and smoke-test are both green. Structurally simple but has its own race/CDN gotchas.
**Delivers:** `publish` job on `ubuntu-latest` with `needs: [build, smoke-test]`; wheel uploaded as GitHub Release asset via `softprops/action-gh-release@v2`; wheel placed at `whl/cu<tag>/llama-cpp-python/`; `index.html` regenerated from directory listing with SHA-256 fragments and `data-requires-python` attrs; `peaceiris/actions-gh-pages@v4 keep_files: true`; `concurrency: group: gh-pages-publish, cancel-in-progress: false`; post-publish HTTP probe loop verifying Fastly shows the new wheel.
**Uses stack:** `peaceiris/actions-gh-pages@v4`, `softprops/action-gh-release@v2`, Python index-generation script.
**Implements architecture:** `publish` job, PEP 503 index format, GitHub Release asset hosting.
**Avoids pitfalls:** #8 (gh-pages concurrent race), #9 (Fastly CDN stale), #17 (PEP 503 name normalization).

### Phase 5: Documentation + consumer UX

**Rationale:** A working index nobody knows how to use has zero value.
**Delivers:** README section with install command and minimum-driver guidance (≥ 551.61 for CUDA 12.4); note on 15-minute Fastly cache delay and `--no-cache-dir` escape hatch; workflow inline comments explaining the `-allow-unsupported-compiler` ban and MSVC 14.39 pin rationale; release notes that record the llama.cpp submodule SHA.
**Avoids pitfalls:** #9 user-facing note, #11 driver-requirement doc.

### Phase 6: Ongoing maintenance + drift detection

**Rationale:** Our fork will face the same `windows-2022` image-rotation pressure that killed upstream. Not required for v1 ship, but cheap insurance.
**Delivers:** Weekly scheduled canary workflow (`cron: '0 6 * * 1'`) that builds with current pins but doesn't publish; `check-upstream-drift.yml` that diffs upstream's `build-wheels-cuda.yaml` monthly and opens an issue; `.github/dependabot.yml` with `package-ecosystem: github-actions`; annotate upstream-inherited YAML blocks with source commit SHA; quarterly check of upstream #2136.
**Avoids pitfalls:** #2 (image rotation canary), #13 (upstream workflow drift), #18 (PR from fork guards).

### Phase Ordering Rationale

- **Strict dependencies:** Phase N+1 cannot be meaningfully exercised without Phase N green. Build → smoke-test → publish is the `needs:` chain literally visible in the YAML.
- **Pitfall-weighted ordering:** The highest-severity pitfalls (#1, #2, #3) all live in Phase 1 — addressing them first collapses the rest of the failure surface. Size discipline (#10) in Phase 2 prevents gh-pages quota surprises in Phase 4.
- **Dev-loop economics:** Cache-on-failure wiring (Phase 2) before the smoke-test job (Phase 3) means "build fails → fix YAML → retry" doesn't pay 30+ minutes per retry.
- **Small-matrix-first:** Phases 1–5 produce a single-combo (3.11 × cu124) working pipeline. Matrix expansion is a post-launch P2 item — jllllll's fork died from matrix overreach.

### Research Flags

**Phases likely needing deeper research during planning:**

- **Phase 1 (scaffold):** MEDIUM-depth — verify `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` is installable on current `windows-2022` image via a throwaway dry-run job.
- **Phase 2 (caching):** LOW-to-MEDIUM — verify CMP0141 + `MSVC_DEBUG_INFORMATION_FORMAT=Embedded` interaction with nvcc's `-Xcompiler` flag passthrough; confirm `GGML_CUDA_FORCE_MMQ=ON` removes the cublas link.
- **Phase 4 (publish):** LOW — well-documented patterns.

**Phases with standard patterns (skip research-phase):**

- **Phase 3 (smoke test):** Smoke-test-in-fresh-venv is established; cuobjdump arch assertion is fully scoped.
- **Phase 5 (docs):** Pure documentation.
- **Phase 6 (maintenance):** Well-documented Dependabot + scheduled workflow patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Every action and version pin cross-referenced to official sources or direct precedent (ExLlamaV2, PyTorch, keepassxc). MEDIUM only on `windows-2025-vs2026` beta (deliberately avoided). |
| Features | HIGH | PEP 503 is authoritative; abetlen's index is directly observable; jllllll's failure is publicly documented. |
| Architecture | HIGH-to-MEDIUM | Three-job DAG, artifact-as-contract, separate publish runner are all standard. MEDIUM on `hashFiles('vendor/llama.cpp/.git/HEAD')` submodule git-file/git-dir edge case — needs first-run verification. |
| Pitfalls | HIGH | Every critical pitfall is backed by a public issue or commit. LOW only on *exact current VS 17.x today* — image churn is time-sensitive. |

**Overall confidence:** HIGH. The convergence across four independent research streams on the same top-10 decisions is the strongest signal. Known unknowns are time-bounded (image churn) not fundamental.

### Gaps to Address

- **`vendor/llama.cpp/.git/HEAD` hashing edge case:** On fresh `submodules: recursive` checkout, `.git` inside the submodule may be a file (git-dir indirection) not a directory. Verify empirically on first Phase 1 run.
- **MSVC 14.39 component availability on current `windows-2022` image:** Microsoft occasionally retires older side-by-side components. Before finalizing Phase 1, run throwaway `vs_installer.exe modify --add ... --dry-run` to confirm. Fallback: chocolatey's `vs-buildtools-vcbuildtools-2022` with pinned version, or switch to CUDA 12.6+ where MSVC 14.40 is in range.
- **sccache + nvcc path-sensitivity:** `SCCACHE_DIRECT=true` and `SCCACHE_IGNORE_SERVER_IO_ERROR=1` avoid most issues, but CUDA-specific sccache behavior on Windows is less battle-tested.
- **Fastly stale-window variance:** Documented as "up to 15 min" but real-world 2–30 min. Probe loop defaults to 15× 60s; extend if flakes.
- **`windows-latest` repointing to `windows-2025`:** Tracked in runner-images #12677. Not affected directly (we pin explicitly), but future phase may want `windows-2025` compat testing.

## Sources

### Primary (HIGH confidence)

- NVIDIA CUDA Installation Guide for Microsoft Windows 12.4.1
- NVIDIA Developer Forums — `_MSC_VER` 1940 + llama.cpp thread
- Microsoft devblogs — "MSVC Toolset Minor Version Number 14.40 in VS 2022 v17.10"
- actions/runner-images Windows2022-Readme.md, issues #10448/#10575/#10954/#12677/#13638
- PEP 503, PEP 427, PEP 440, PEP 658, PEP 691, PEP 740
- PyPA name normalization spec
- peaceiris/actions-gh-pages README, hendrikmuhs/ccache-action README, actions/cache v4/v5 READMEs
- abetlen/llama-cpp-python issues #1543, #1551, #1838, #1894, #2068, #2127, #2136
- abetlen/llama-cpp-python commits `98fda8c`, `4f17ae5`
- Jimver/cuda-toolkit issue #382
- Microsoft/STL issue #4941
- cupy/cupy #8578, nerfstudio-project/nerfstudio #3157/#3171
- PyTorch issue #156181

### Secondary (MEDIUM confidence)

- Quasar CUDA/MSVC compatibility matrix
- ccache/ccache issue #1040, Mozilla Bugzilla 1000726
- abetlen's gh-pages index (direct UX observation)
- PyTorch wheel index (variant-path convention)
- jllllll archived index (cautionary matrix-overreach example)

### Tertiary (LOW confidence)

- MSVC 14.39 component retirement date on GitHub's `windows-2022` image — not published
- Fastly CDN stale window upper bound — "up to 10 min" documented, up to 30 min anecdotal
- CUDA 12.8+ `_MSC_VER` ceiling — no authoritative recent source

### Local references

- `C:\claude_checkouts\llama-cpp-python\.github\workflows\build-wheels-cuda.yaml`
- `C:\claude_checkouts\llama-cpp-python\.planning\user_specs\001_WINDOWS_CUDA_WHEELS_CI_PLAN.md`
- `C:\claude_checkouts\llama-cpp-python\.planning\PROJECT.md`
- `C:\claude_checkouts\llama-cpp-python\.planning\codebase\ARCHITECTURE.md`
- `C:\claude_checkouts\keepassxc\.github\workflows\build.yml`

---
*Research completed: 2026-04-15*
*Ready for roadmap: yes*
