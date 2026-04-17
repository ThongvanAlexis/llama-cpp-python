# Phase 3: Smoke Test (Publish Gate) - Context

**Gathered:** 2026-04-17
**Status:** Ready for planning

<domain>
## Phase Boundary

A separate `smoke-test` job on a fresh `windows-2022` runner, sparse-checked-out to exclude `llama_cpp/` source, installs the built wheel into a clean venv and proves it loads a 27 KB GGUF fixture and generates 2 tokens without crashing; the publish job declares `needs: [build, smoke-test]` so a red smoke test is structurally incapable of reaching gh-pages. Phase 3 adds the smoke-test job to the existing workflow file; the publish job itself is Phase 4.

</domain>

<decisions>
## Implementation Decisions

### cuobjdump sourcing
- Install full `cuda-toolkit` via mamba on the smoke-test runner (same `conda-incubator/setup-miniconda@v3.1.0` + `mamba install cuda-toolkit` pattern as the build job)
- cuobjdump is guaranteed present with the full toolkit; no need for minimal subset
- Inspect `ggml-cuda.dll` in site-packages AFTER `pip install` (natural order — wheel is already installed for the import test, DLL is unpacked in `site-packages/llama_cpp/lib/`)

### Sparse checkout scope
- Full checkout MINUS `llama_cpp/` directory — use sparse-checkout to exclude only the source package directory
- Must include `llm_for_ci_test/` (GGUF fixture), `.github/workflows/` (workflow file itself), and everything else
- Explicit assertion: `Test-Path llama_cpp/__init__.py` must FAIL (proves sparse checkout excluded source)
- Explicit assertion: `Test-Path llm_for_ci_test/tiny-llama.gguf` must PASS (proves fixture survived sparse checkout)
- After `import llama_cpp`, assert `llama_cpp.__file__` contains `site-packages` and does NOT contain the repo checkout path — proves ST-03's intent

### Gate assertion placement
- The `needs: [build, smoke-test]` grep-assert belongs in the `lint-workflow` job (alongside the existing ban-grep step)
- Uses exact string match: grep for literal `needs: [build, smoke-test]` (tight — catches reordering or removal)
- **Deferred to Phase 4**: the assertion only makes sense when the publish job exists; Phase 3 does NOT add a stub publish job or a skip-until-Phase-4 assertion
- Phase 3's deliverable is the smoke-test job itself; the structural gate enforcement is Phase 4's responsibility

### Failure diagnostics
- Full forensics on BOTH green and red paths (matching Phase 1's `$GITHUB_STEP_SUMMARY` pattern):
  - **Green summary**: table with wheel filename, installed version, import path, cuobjdump arch list, venv Python path, inference exit code, output tokens
  - **Red summary**: venv layout (`pip list`), Python path, DLL search path, import resolution trace, ggml-cuda.dll location, cuobjdump output (if available), exception traceback
- Capture the model's 2 output tokens in the log (output is gibberish from a 27KB model, but seeing it proves the full inference pipeline ran)
- Run inference test in a **subprocess** (`python -c '...'`) — segfaults/access violations produce a non-zero exit code that pwsh catches cleanly, rather than killing the step's process
- **Disable Windows Error Reporting (WER) crash dialog** via registry keys (`DontShowUI=1`, `Disabled=1`) — prevents a modal crash dialog from hanging the CI runner indefinitely on access violation
- **Cross-check installed version**: after `pip install`, verify `llama_cpp.__version__` contains the expected `+cu126.ll<sha>` pattern — catches wrong-wheel-installed scenarios and proves the version embed from Phase 2 survives installation

### Claude's Discretion
- Exact sparse-checkout configuration syntax (cone mode vs no-cone, actions/checkout sparse-checkout input vs manual git sparse-checkout)
- Step ordering within the smoke-test job (setup mamba first or download artifact first — whichever is most efficient)
- Exact WER registry key paths and values
- venv creation mechanism (`python -m venv` vs virtualenv)
- Exact Python subprocess invocation for the inference test
- Per-step `timeout-minutes` values
- Forensics summary table layout and column order
- Remediation hint wording on assertion failures
- Whether to cache mamba pkgs on the smoke-test runner (probably yes — same `if: always()` save pattern)

</decisions>

<specifics>
## Specific Ideas

- The `llm_for_ci_test/tiny-llama.gguf` file already exists in the repo (observed during codebase scout). Verify it's ~27 KB and loadable before considering ST-01 done.
- The build job uploads the wheel artifact as `cuda-wheel` — this is the download contract (confirmed in Phase 2 STATE.md).
- The inference test command from the requirements: `Llama('llm_for_ci_test/tiny-llama.gguf', n_ctx=64)('hi', max_tokens=2)` — run in subprocess, capture output.
- Expected CUDA architectures for cuobjdump: `sm_80`, `sm_86`, `sm_89`, `sm_90` (from CUDA archs `80-real;86-real;89-real;90-real;90-virtual` in Phase 2 CONTEXT.md).
- The smoke-test job should NOT have `needs: [build]` silently — it should explicitly depend on `build` via `needs: [build]` so the job DAG is clear.
- SC-5 (deliberately-broken wheel causes red) is an acceptance criterion, not something the workflow implements. It's verified by actually dispatching a broken build.

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- **Build job mamba setup pattern**: `conda-incubator/setup-miniconda@v3.1.0` with `miniforge-version: latest` + `mamba install cuda-toolkit=$cudaVersion` — reuse identically for smoke-test runner
- **lint-workflow job**: existing ban-grep + actionlint steps — Phase 4 will add the `needs:` grep-assert here
- **`actions/download-artifact@v4`**: standard GH Actions pattern for consuming build artifacts in downstream jobs
- **probe-msvc step**: NOT needed on smoke-test runner (no compilation happens)
- **Existing `llm_for_ci_test/tiny-llama.gguf`**: already committed, ~27 KB GGUF fixture

### Established Patterns
- All Windows steps use `shell: pwsh` with `Set-StrictMode -Version Latest; $ErrorActionPreference = 'Stop'` prelude (Phase 1 convention)
- External actions pinned with `@v4`/`@v5` semantic tags (Phase 1)
- Hard-fail with remediation hints on any assertion failure (Phase 1 pattern)
- `if: always()` for cache saves (Phase 2 pattern)
- `if: success()` for green forensics summary, `if: failure()` for red diagnostics (Phase 1 pattern)

### Integration Points
- `needs: [build]` — smoke-test job depends on build job completing successfully
- `actions/download-artifact@v4` with `name: cuda-wheel` — downloads the wheel from build job
- `ggml-cuda.dll` lives at `site-packages/llama_cpp/lib/ggml-cuda.dll` after pip install (from CMakeLists.txt's `TARGET_RUNTIME_DLLS` copy)
- Phase 4's publish job will declare `needs: [build, smoke-test]` to complete the gate

</code_context>

<deferred>
## Deferred Ideas

- **`needs: [build, smoke-test]` grep-assert in lint-workflow** — deferred to Phase 4 when the publish job is created. Phase 3 adds the smoke-test job; Phase 4 adds the publish job + the structural gate assertion.
- **Multi-platform smoke test** — Linux/macOS smoke tests for other wheel builds (v2 MX-* scope).
- **GPU-accelerated inference test** — would require a GPU runner. Current test runs on CPU-only runner; cuobjdump compensates by verifying CUDA arch presence in the DLL. GPU testing is v2 scope.

</deferred>

---

*Phase: 03-smoke-test-publish-gate*
*Context gathered: 2026-04-17*
