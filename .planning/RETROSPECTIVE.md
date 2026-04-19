# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 — Windows CUDA Wheels CI

**Shipped:** 2026-04-19
**Phases:** 4 | **Plans:** 8 | **Timeline:** ~5 days (2026-04-15 → 2026-04-19)

### What Was Built

- A single GitHub Actions workflow file (`.github/workflows/build-wheels-cuda-windows.yaml`, 1842 lines) with a four-job DAG: `build → smoke-test → publish`, plus `lint-workflow` in parallel.
- Live GitHub Release `v0.3.20-cu126-win` with a runtime-verified wheel (`llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl`, 356 MB) and DOC-05-compliant forensics body.
- Fork-focused `README.md` with manual-download install flow (`pip install path/to/wheel.whl`); upstream README preserved byte-for-byte at `README.upstream.md`.

### What Worked

- **Empirical dispatch loop as a first-class checkpoint.** Phase 2 Plan 02 and Phase 3 Plan 01 were both explicitly `human-verify` tasks that expected multiple dispatch iterations. Treating "run the real workflow, read the logs, fix, re-dispatch" as a supported wave (not a deviation) surfaced 16+ bugs that would have been invisible to static analysis — scikit-build-core silently dropping `CMAKE_BUILD_TYPE=Release`, sccache daemon OOM on parallel nvcc, `GGML_NATIVE=ON` baking `/arch:AVX512` from AVX-512 VMs, the `-C wheel.py-api=""` vs env-var asymmetry.
- **Regex-char-class self-match-safe greps.** `allow[-]unsupported[-]compiler` for the ban-grep and `^    needs[:]` for the publish-gate presence-grep both prevented the lint-workflow step from matching its own source line. Caught Phase 1 Plan 01's ban-grep self-match bug (commit `23ce327`) and Phase 4 Plan 01's unanchored-pattern 6-match bug (Deviation 1) before either shipped.
- **Structural gates over assertion gates.** Making publish `needs: [build, smoke-test]` (DAG-level) and then adding a lint-grep that asserts that line exists gives two independent guarantees: a red smoke-test is structurally incapable of reaching the Release API, AND deletion of the dependency is grep-detectable in the next dispatch.
- **User-approved scope reductions mid-flight.** Two significant shifts landed cleanly: (1) Phase 3 simplified from CPU inference on a GGUF fixture to a DLL-load import gate after the user pointed out their real workload uses `n_gpu_layers=-1` and never exercises CPU hot paths; (2) Phase 4 dropped gh-pages PEP 503 indexing (PUB-03..10 + DOC-03) because the maintenance cost didn't justify the complexity for a solo-maintainer fork. Both were documented in REQUIREMENTS.md and PROJECT.md Key Decisions the same day.

### What Was Inefficient

- **scikit-build-core `CMAKE_BUILD_TYPE` wrestling match.** Seven commits to discover that every documented override channel is silently ignored (pyproject.toml TOML key, `-C cmake.build-type=Release`, `SKBUILD_CMAKE_BUILD_TYPE` env var, `-DCMAKE_BUILD_TYPE=Release` in `CMAKE_ARGS`). The workaround — `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` + `CMP0091=NEW` + `CMAKE_*_FLAGS_DEBUG` overrides — is sound but took ~2 days of dispatch iteration to find. Research phase could have flagged this upfront by reading scikit-build-core source, not just docs.
- **Phase 3 VERIFICATION.md never written.** The phase shipped working end-to-end (two independently green dispatches) but the formal verification document was skipped in the heat of the iteration loop. Caught only at milestone audit. Process gap, not a functional gap; cost ~1 hour of audit reconciliation.
- **REQUIREMENTS.md traceability drift.** ST-01..ST-08 checkboxes stayed `[ ]` through milestone close even though 5 were satisfied, 2 superseded, 1 delivered by Phase 4. ST-01 and ST-06 supersessions were only documented in Phase 3 SUMMARY — the authoritative REQUIREMENTS.md was silent. Pattern: SUMMARY captures user-approved scope changes; REQUIREMENTS.md isn't always updated the same commit. Fix for v2: treat REQUIREMENTS.md edits as part of the supersession commit, not a follow-up.

### Patterns Established

1. **Preflight-first topology.** Toolchain assertions run BEFORE checkout/python/mamba/build so misconfigured inputs hard-fail within ~5s with remediation hints instead of consuming 30+ min of wasted build time.
2. **Regex char classes for self-match-safe greps.** Any grep-assert that targets its own workflow file should wrap literal patterns in `[-]` char classes (or equivalent) so the pattern never matches its own source.
3. **Paraphrased diagnostic strings.** Error messages and comment prose that reference banned flags use paraphrases ("compiler-bypass workaround flag", "colon"/"dash" for YAML syntax) to stay regex-invisible for ban-greps and count-equals-1 invariants.
4. **CI-time pyproject.toml patching with `git checkout --` restore.** Same restore-pattern as the `__init__.py` version embed — keeps the tracked file clean so upstream merges remain painless. Used in Phase 2 (version embed) and Phase 3 (scikit-build-core compat-mode exit + build-type injection).
5. **Forensics summary + remediation hint paired steps.** Every phase wrote both a green `$GITHUB_STEP_SUMMARY` table and a red remediation-hint step. Run-meta JSON + publish-log.txt + release.json persisted as commit-tracked forensics under phase directories.
6. **pefile auto-discovery for synthesized stubs.** Rather than hand-maintaining a symbol list that breaks whenever llama.cpp adds a new driver-API call, scan both toolkit DLLs and wheel DLLs for imports, synthesize the stub programmatically. Forward-compatible without churn.

### Key Lessons

1. **scikit-build-core is an opinionated black box.** It strips/rewrites CMake invocations in ways that are only documented in source. For CRT/build-type control, bypass via `CMAKE_MSVC_RUNTIME_LIBRARY` + `CMAKE_*_FLAGS_DEBUG` overrides — these DO pass through.
2. **GitHub-hosted Windows runners have non-deterministic CPU SKUs.** Don't let llama.cpp's `FindSIMD.cmake` auto-detect (`GGML_NATIVE=ON`). Always set `GGML_NATIVE=OFF` and declare flags explicitly for portable wheels.
3. **Driverless CI runners need a synthesized nvcuda.dll.** Otherwise `DllMain` of `ggml-cuda.dll` fails to resolve driver-API functions and the whole DLL chain collapses before your test code runs. pefile auto-discovery + `cl /O2 /LD` compile is reliable.
4. **Documentation lag kills audit throughput.** Every user-approved scope change should update REQUIREMENTS.md in the same commit as the code change, not in a later SUMMARY-writing commit. ST-01/ST-06 supersessions cost ~1 hour of audit reconciliation because they were only captured in SUMMARY.
5. **Empirical dispatches are first-class work, not deviations.** Phase 2 and Phase 3 both took multi-session fix-loops against real GitHub Actions runs. Planning should budget for these explicitly rather than treat each fix iteration as a plan violation.

### Cost Observations

- **Sessions:** multi-session across ~5 days; Phase 1 Plan 02 alone was ~24h of multi-session empirical work.
- **Dispatch fix iterations:** 16+ commits in Phase 3 Plan 01 alone, 6+ in Phase 2 Plan 02. Most were sub-$0.10 — small surgical edits to CMAKE_ARGS or cache keys — but the cadence was slow because each dispatch costs ~120 min of runner time to surface the next failure.
- **Notable:** Model profile `quality` (per config.json) was consistently routed to Opus for research / planning / verification agents; this paid off on TC-02 `_MSC_VER` compat-matrix research (caught the per-generation cap model corrected on 2026-04-16) and on the Phase 3 supersession decision (clean framing of "what does this phase really gate?").

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Phases | Plans | Key Change |
|-----------|--------|-------|------------|
| v1.0 | 4 | 8 | Empirical dispatch-verification treated as a first-class plan wave; regex-char-class greps adopted as a pattern; user-approved mid-flight scope reductions documented in PROJECT.md Key Decisions same-day. |

### Cumulative Quality

| Milestone | Requirements Active | Satisfied | Partial | Superseded (user-approved) |
|-----------|---------------------|-----------|---------|----------------------------|
| v1.0 | 42 | 39 (+ 1 via another phase) | 1 | 2 |

### Top Lessons (Verified Across Milestones)

1. **Empirical dispatch is the truth; static review is a hint.** (v1.0 — 16+ dispatch-discovered bugs across Phase 2/3/4 that passed static review.)
2. **Any grep-assert targeting its own file needs regex char classes.** (v1.0 — caught twice: Phase 1 Plan 01 ban-grep, Phase 4 Plan 01 publish-gate presence-grep.)
3. **Document user-approved scope changes in REQUIREMENTS.md the same commit, not in the SUMMARY later.** (v1.0 — ST-01/ST-06 supersession debt cost ~1 hour of audit reconciliation.)
