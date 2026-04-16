# Roadmap: Windows CUDA Wheels CI for llama-cpp-python (fork)

## Overview

This v1 delivers a GitHub Actions workflow that produces a runtime-verified Windows x64 CUDA wheel of `llama-cpp-python` and publishes it via a PEP 503 pip index on `gh-pages`, so `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu126` Just Works. The roadmap is shaped by the single non-negotiable architectural invariant that defines the project: **publish is gated on smoke test in a separate job with a hard `needs:` dependency** — the whole reason this fork exists is to never repeat upstream's failure mode of shipping wheels that compile but segfault at runtime (issue #1543, caused by `-allow-unsupported-compiler` bypassing nvcc's `_MSC_VER` check). Phase 1 fights the VS/nvcc toolchain battle first because every later investment is wasted until that is green; Phase 2 adds the build body and caches (save-on-failure because the dev loop is mostly failing builds); Phase 3 is the smoke-test publish gate; Phase 4 is publish + consumer UX (PEP 503 index, README, release notes). Granularity is coarse per `config.json` (3-5 phases, 1-3 plans each); the research SUMMARY's proposed Phase 6 (maintenance / canary / drift-check) is explicitly out of v1 scope — it maps to `AUT-*` items in v2.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3, 4): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [x] **Phase 1: Scaffold & Toolchain Pinning** - New Windows-only workflow file with MSVC auto-select from CUDA compat matrix and CUDA toolkit unified behind one install path (completed 2026-04-16)
- [ ] **Phase 2: Build & Cache** - scikit-build-core produces a correctly-tagged, size-bounded wheel with three-tier save-on-failure caches
- [ ] **Phase 3: Smoke Test (Publish Gate)** - Fresh-runner job installs the wheel and asserts `Llama(...)` runs without segfault; publish blocked if red
- [ ] **Phase 4: Publish & Consumer UX** - PEP 503 index on gh-pages, release assets, post-publish probe, and README install docs

## Phase Details

### Phase 1: Scaffold & Toolchain Pinning
**Goal**: A `workflow_dispatch`-only Windows workflow file exists that, when dispatched with default inputs, reliably activates MSVC 14.40 and CUDA 12.6.3 on a `windows-2022` runner, with preflight assertions that fail loudly (rather than silently producing segfaulting wheels) when the toolchain is out of band. (Originally spec'd for 14.39 + 12.4.1; bumped 2026-04-15 after OQ1 resolved — VC.14.39.17.9 component removed from windows-2022 per actions/runner-images#9701.)
**Depends on**: Nothing (first phase)
**Requirements**: WF-01, WF-02, WF-03, WF-04, WF-05, TC-01, TC-02, TC-03, TC-04, TC-05, TC-06, TC-07, TC-08, TC-09, TC-10, DOC-04
**Success Criteria** (what must be TRUE):
  1. A dispatcher can run `build-wheels-cuda-windows.yaml` from the Actions UI with `python_version=3.11` and `cuda_version=12.6.3` defaults, and the workflow reaches the preflight checks without YAML/trigger errors.
  2. On a green preflight run the job log shows `cl.exe /Bv` reporting `_MSC_VER <= 1940` (cap computed from `msvc_toolset=14.40` via `1900 + minor`) AND `nvcc --version` matches the dispatched `cuda_version`; if either is out of range the job fails with a clear named-step error (not a silent downstream compile failure).
  3. Grepping the workflow file for the literal string `allow-unsupported-compiler` returns zero matches, and a CI self-check step enforces this.
  4. Only one CUDA install path is active in the job (mamba `cuda-toolkit`); `where nvcc` returns exactly one path and no stray `CUDA_PATH_V*_*` env variables survive into the build step.
  5. The VS BuildCustomizations directory is discovered dynamically and the workflow fails loudly (strict-mode) if no VS 2022 install is found, rather than silently no-op'ing.
**Plans**: 3 plans

Plans:
- [x] 01-01-PLAN.md — Create workflow skeleton + operational lint-workflow job (WF-01, WF-02, WF-03, WF-04, TC-01, TC-10)
- [x] 01-02-PLAN.md — Populate preflight body: toolchain install + 5 named assertions + VS discovery (WF-05, TC-02..TC-09) [human-verify checkpoint]
- [x] 01-03-PLAN.md — Forensics summary (green/red paths) + DOC-04 inline comments (DOC-04) [human-verify checkpoint]

### Phase 2: Build & Cache
**Goal**: The build step, invoked after Phase 1's green toolchain, produces a correctly-tagged `cp311-cp311-win_amd64.whl` under 400 MB with a reproducible version string (embeds llama.cpp submodule SHA), while caches (sccache, mamba pkgs) are keyed to invalidate on source/toolchain change and save on failure so the fail-fix-retry loop doesn't re-pay cold costs.
**Depends on**: Phase 1
**Requirements**: BLD-01, BLD-02, BLD-03, BLD-04, BLD-05, BLD-06, BLD-07, BLD-08, BLD-09, BLD-10, BLD-11, BLD-12, BLD-13
**Success Criteria** (what must be TRUE):
  1. A green build run produces exactly one `dist/llama_cpp_python-<ver>+cu126.ll<sha>-cp311-cp311-win_amd64.whl` where `<sha>` is the llama.cpp submodule short SHA; a regex guard on the wheel filename fails the job on any drift to `abi3`, `none-any`, or a wrong Python tag.
  2. The produced wheel is strictly under 400 MB (CI asserts `wheel.size < 400 MiB` and fails if exceeded); the wheel is uploaded as an Actions artifact consumable by downstream jobs.
  3. On a rebuild after a YAML edit (no source change), `sccache --show-stats` reports a cache hit rate above 30% and the build step wall time is meaningfully shorter than the cold run, proving `/Z7` + `CMP0141=NEW` took effect.
  4. After a deliberately-failing build (e.g., introduced compile error), every cache (sccache, mamba pkgs dir) is still saved under its key and restores cleanly on the next attempt.
  5. Each job step has an explicit `timeout-minutes` so no single hung step can eat the entire 6-hour runner budget.
**Plans**: 2 plans

Plans:
- [ ] 02-01-PLAN.md — Build body + cache wiring: vcvarsall activation, CMAKE_ARGS (Ninja, sccache, /Z7, CUDA archs), version override with SHA, mamba split cache with save-on-failure, sccache setup, per-step timeouts (BLD-01..09, BLD-12)
- [ ] 02-02-PLAN.md — Post-build assertions + artifact upload + dispatch verification: wheel tag regex guard, size check, upload-artifact, end-to-end dispatch [human-verify checkpoint] (BLD-10, BLD-11, BLD-13)

### Phase 3: Smoke Test (Publish Gate)
**Goal**: A separate `smoke-test` job on a fresh `windows-2022` runner, sparse-checked-out to exclude `llama_cpp/` source, installs the built wheel into a clean venv and proves it loads a 27 KB GGUF fixture and generates 2 tokens without crashing; the publish job declares `needs: [build, smoke-test]` so a red smoke test is structurally incapable of reaching gh-pages.
**Depends on**: Phase 2
**Requirements**: ST-01, ST-02, ST-03, ST-04, ST-05, ST-06, ST-07, ST-08
**Success Criteria** (what must be TRUE):
  1. `llm_for_ci_test/tiny-llama.gguf` (27 KB) exists in the repo HEAD and is fetched by the smoke-test job's sparse checkout.
  2. A green smoke-test log shows `pip install <wheel>.whl` in a freshly-created venv, `from llama_cpp import Llama` resolving to `site-packages` (not to `./llama_cpp/`), and `Llama('llm_for_ci_test/tiny-llama.gguf', n_ctx=64)('hi', max_tokens=2)` exiting zero with no Access Violation dialog or segfault.
  3. The smoke-test job runs `cuobjdump --list-elf ggml-cuda.dll` and fails if the expected CUDA architectures (`sm_80`, `sm_86`, `sm_89`, `sm_90`) are not all present, compensating for the absence of a GPU on the runner.
  4. Deleting `smoke-test` from `needs:` on the publish job is a grep-detectable regression: the workflow contains `needs: [build, smoke-test]` on the publish job and a CI self-check asserts it, so publish is structurally gated on smoke test.
  5. A deliberately-broken wheel (e.g., one built with `-allow-unsupported-compiler` in a spiked branch) causes the smoke-test job to go red, and the publish job correctly does not run for that dispatch.
**Plans**: TBD (1-3 plans; likely 1 plan — smoke-test + fixture + gate-assertion cluster together)

Plans:
- [ ] 03-01: TBD — commit tiny-llama.gguf fixture, add smoke-test job with sparse checkout + fresh venv + Llama inference + cuobjdump arch check + `needs:` gate

### Phase 4: Publish & Consumer UX
**Goal**: On green smoke-test, the publish job (on `ubuntu-latest`) uploads the wheel as a GitHub Release asset, places it under `whl/cu126/llama-cpp-python/` on the `gh-pages` branch with a regenerated PEP 503-compliant `index.html` (SHA-256 fragments + `data-requires-python` attrs), guarded by a `concurrency:` block and a post-publish HTTP probe — plus the consumer-facing README updates that make the index usable.
**Depends on**: Phase 3
**Requirements**: PUB-01, PUB-02, PUB-03, PUB-04, PUB-05, PUB-06, PUB-07, PUB-08, PUB-09, PUB-10, DOC-01, DOC-02, DOC-03, DOC-05
**Success Criteria** (what must be TRUE):
  1. After a green publish run, `curl https://<user>.github.io/llama-cpp-python/whl/cu126/llama-cpp-python/` returns a PEP 503-valid HTML index whose `<a>` entries include `#sha256=<digest>` fragments and `data-requires-python` attributes, and the just-published wheel filename appears among them.
  2. Running `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu126` in a fresh venv (possibly with `--no-cache-dir` during the 15-minute Fastly window) installs the new wheel without `No matching distribution` errors, and the post-publish probe step confirms Fastly visibility before the publish job exits green.
  3. After a second dispatch (different Python or CUDA version), the previously-published wheel is still listed in the index alongside the new one — `keep_files: true` and `concurrency: group: gh-pages-publish, cancel-in-progress: false` prevent the race that would wipe it.
  4. The published GitHub Release body records the llama.cpp submodule SHA at build time; the wheel is present as a release asset (authoritative storage) and at its gh-pages path (pip-facing copy).
  5. The repo's README contains an "Install (Windows CUDA)" section with the canonical `--extra-index-url` command, the minimum NVIDIA driver version (≥ 551.61 for CUDA 12.4), and a note on the up-to-15-minute Fastly cache delay plus the `--no-cache-dir` escape hatch.
**Plans**: TBD (1-3 plans; likely 2-3 — publish job, index generator, README/release-notes are distinct concerns)

Plans:
- [ ] 04-01: TBD — publish job on ubuntu-latest, Release asset upload, gh-pages placement, PEP 503 index regen with SHA-256 + requires-python, concurrency block, post-publish probe
- [ ] 04-02: TBD — README install section (canonical command, driver floor, Fastly delay note) + release notes template embedding llama.cpp SHA

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Scaffold & Toolchain Pinning | 3/3 | Complete | 2026-04-16 |
| 2. Build & Cache | 0/2 | Planned | - |
| 3. Smoke Test (Publish Gate) | 0/TBD | Not started | - |
| 4. Publish & Consumer UX | 0/TBD | Not started | - |

## Coverage

**v1 requirements mapped:** 51 / 51 ✓
**Orphaned requirements:** 0
**v2 requirements (deferred):** MX-01, MX-02, MX-03, AUT-01, AUT-02, AUT-03, AUT-04, AUT-05, QA-01, QA-02, QA-03, QA-04 — tracked in REQUIREMENTS.md, not in v1 roadmap

**Granularity:** coarse (config.json) → 4 phases within the 3-5 band. Research SUMMARY proposed 6; compressed by (a) cutting Phase 6 (maintenance/canary/drift-check) entirely — v2 scope, maps to `AUT-*` items, and (b) folding docs into Phase 4 where the consumer UX belongs (DOC-04 the YAML inline-comment requirement goes to Phase 1 where the YAML is actually written).

---
*Roadmap created: 2026-04-15*
*Phase 1 planned: 2026-04-15 (3 plans, wave-sequential)*
*Updated 2026-04-15 — OQ1 resolution: default cuda_version 12.4.1 → 12.6.3, msvc_toolset 14.39 → 14.40 (VC.14.39.17.9 component retired from windows-2022 image per actions/runner-images#9701)*
*Phase 2 planned: 2026-04-16 (2 plans, wave-sequential — build body + cache in wave 1, assertions + dispatch verification in wave 2)*
