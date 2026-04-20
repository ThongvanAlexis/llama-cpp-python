# Phase 1: Scaffold & Toolchain Pinning - Context

**Gathered:** 2026-04-15
**Status:** Ready for planning

<domain>
## Phase Boundary

A `workflow_dispatch`-only Windows workflow file (`.github/workflows/build-wheels-cuda-windows.yaml`) exists that, when dispatched with default inputs, activates MSVC 14.39 and CUDA 12.4.1 on a `windows-2022` runner and runs preflight assertions that fail loudly (with remediation hints) when the toolchain is out of band. Phase 1 ends at "preflight green on default dispatch"; no `build`, `smoke-test`, or `publish` jobs exist yet — those are added by Phases 2, 3, and 4 respectively.

</domain>

<decisions>
## Implementation Decisions

### Job layout

- Phase 1 YAML contains exactly two jobs: `preflight` (Windows) and `lint-workflow` (Ubuntu, runs in parallel).
- No stubbed `build` / `smoke-test` / `publish` job shells. Each future phase adds its own job.
- A separate `prepare-toolchain` *step* inside the `preflight` job handles VS/MSVC install and `vswhere` probing; preflight assertion steps are strictly read-only ("assert, don't mutate").

### Preflight scope & structure

- Assertions (one step per assertion, named explicitly so failures are pinpointable in the Actions UI):
  1. MSVC toolset → `cl.exe /Bv` reports `_MSC_VER ≤ ${msvc_toolset_msc_cap}` (cap derived from `msvc_toolset` input, not hardcoded 1939).
  2. CUDA toolkit → `nvcc --version` matches the dispatched `cuda_version` input.
  3. Single nvcc path → `where nvcc` returns exactly one path AND no `CUDA_PATH_V*_*` env survives into preflight (TC-07 asserted here, not just hoped for at build time).
  4. Submodule populated → `vendor/llama.cpp/CMakeLists.txt` exists (catches silent `submodules: recursive` failures from WF-05).
  5. Python version match → `python --version` matches the dispatched `python_version` input (catches setup-python cache misses or wrong interpreter on PATH).
- On green preflight, emit a `$GITHUB_STEP_SUMMARY` markdown table with: MSVC version, `_MSC_VER`, nvcc version, CUDA path, Python version, llama.cpp submodule short SHA, runner OS build number, detected VS install path, mamba version, disk free space, `$env:PROCESSOR_IDENTIFIER`. Purpose: forensics when a wheel later misbehaves.
- On red preflight, emit actionable remediation hints (not just raw assertion output). Example: if `_MSC_VER > ${cap}`, hint reads roughly: "Runner VS image rotated. Retry dispatch with `msvc_toolset: 14.XX` matching the runner, or verify nvcc [version] compatibility per upstream #1543." Assertion failures should never require a human to cross-reference upstream issues from memory.

### Dispatch inputs

- `python_version` → `choice`, default `3.11`, options `['3.10', '3.11', '3.12']`. 3.13 excluded (MX-03 / v2). Freeform rejected — dropdown makes valid set self-documenting in the Actions UI.
- `cuda_version` → `choice`, default `12.4.1`, options `['12.4.1']` (single-item). Expanding to 12.6.1 is MX-02 / v2. Single-item dropdown makes the v1 constraint visible.
- `msvc_toolset` → string-ish input, default `14.39`. Escape hatch for runner image rotation. **The preflight `_MSC_VER` cap must be computed from this input, not hardcoded 1939** — bumping the input to `14.40` must also bump the cap to `1940`, `14.41` → `1941`, etc.
- `verbose` → boolean, default false. Toggles `$env:VERBOSE=1` and step tracing for debugging preflight steps.
- `dry_run` → boolean, default false. When true, preflight exits green without invoking downstream jobs (useful once Phase 2 adds the build body — saves a full cold build cycle when only debugging toolchain pinning).
- Drift behavior: hard-fail on any mismatch with remediation hint. No warn-and-continue path; that is the upstream #1543 failure mode.

### MSVC 14.39 install strategy

- **Probe first, install second**: call `vswhere -products * -requires Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` before attempting `vs_installer.exe modify`. If `vswhere` returns empty, fail immediately with remediation hint — don't waste runner minutes on a slow installer call that will fail.
- If `vswhere` finds the component, invoke `vs_installer.exe modify --add Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64 --quiet --norestart` (the component ID is actually derived from the `msvc_toolset` input).
- **No auto-fallback.** If install fails, hard-fail with remediation hint pointing at the `msvc_toolset` dispatch input. Explicitly rejected: chocolatey `vs-buildtools` fallback (silent ABI drift), auto-downgrade to nearest installed 14.x (same class of error as `-allow-unsupported-compiler`).
- `ilammy/msvc-dev-cmd@v1` is pinned to `vs-version: '[17.9,17.10)'`. VS 17.10+ on the runner trips preflight and forces re-validation before we ship — tight pin is the whole point.

### Ban-grep self-check (TC-10)

- Separate `lint-workflow` job, runs on `ubuntu-latest` in parallel with `preflight` (free Ubuntu minutes, runs in seconds, shows as independent red/green check on the dashboard).
- Scope: grep `.github/workflows/build-wheels-cuda-windows.yaml` ONLY. Do not grep upstream's `build-wheels-cuda.yaml` (it legitimately contains the flag; we don't own it). Do not grep `CMakeLists.txt` / `pyproject.toml` (out of phase scope; TC-10 is about the workflow file). Do not grep repo-wide.
- Implementation: `grep -nH 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml && exit 1 || exit 0`.
- Phase 3's publish job will later declare `needs:` on this job too (structural gate against regressions).

### File organization

- Single YAML file. No composite actions under `.github/actions/`, no external PowerShell scripts, no reusable workflow fragments. Matches PROJECT.md constraint: *"GitHub Actions YAML only. No shell scripts outside the workflow (keep debugging surface small)."*
- All preflight assertions are inline PowerShell in `run:` blocks (`shell: pwsh`).

### YAML documentation (DOC-04)

- Inline comments explain: (1) why MSVC 14.39 is pinned (nvcc's `_MSC_VER` check, link to upstream #1543), (2) why `-allow-unsupported-compiler` is banned (runtime-segfault trap), (3) why VS range is `[17.9,17.10)` not looser, (4) what `msvc_toolset` dispatch input is for (emergency mitigation when runner image rotates).

### Claude's Discretion

- Exact remediation-hint wording (just make it actionable and unambiguous; include upstream issue numbers where relevant).
- Forensics summary table column order and formatting in `$GITHUB_STEP_SUMMARY`.
- Exact PowerShell idioms for assertions (`Test-Path`, `Select-String`, `throw`, `exit 1` — whatever reads cleanest).
- Mechanism for setting `LongPathsEnabled=1` (TC-09) before checkout — whichever of `reg add` / `New-ItemProperty` / `actions/setup-` prelude is idiomatic.
- Mechanism for unsetting `CUDA_PATH_V*_*` env vars (TC-07) — loop through `Get-ChildItem env:` matching the pattern and `Remove-Item`.
- Step-level `timeout-minutes` values (BLD-09 is Phase 2, but Phase 1 should still bound its steps — reasonable defaults).
- Whether the `prepare-toolchain` step emits a "starting VS component install" progress line (minor log polish).

</decisions>

<specifics>
## Specific Ideas

- "Fail loudly, never silent drift" is the guiding principle. Every decision above leans toward hard-fail-with-remediation over warn-and-continue. The entire reason this fork exists is that upstream's `-allow-unsupported-compiler` shortcut produced wheels that compile but segfault at runtime (#1543). Phase 1's preflight is the first line of defense against that class of error.
- Forensics matter: when a wheel built today segfaults for a user six weeks from now, the `$GITHUB_STEP_SUMMARY` table from its build must be enough to reconstruct the exact toolchain. Err on the side of more columns, not fewer.
- The `msvc_toolset` dispatch input trades a small amount of input-surface complexity for operational resilience: when a runner image rotation breaks 14.39 at 3am, the dispatcher can pivot to 14.40 in the Actions UI without opening a PR. The preflight-cap-derived-from-input pattern (`14.39 → 1939`, `14.40 → 1940`) is the load-bearing detail that makes this safe.

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets

- **Upstream `build-wheels-cuda.yaml`**: ~80% of the Windows steps are already written but gated behind a commented `# 'windows-2022'` in the OS matrix. Useful as a *reference* for the Jimver CUDA zip fetch pattern, mamba install invocation, and VS BuildCustomizations copy idiom — but this phase's deliverable is a NEW file, not edits to that one (WF-01, merge-conflict hygiene). Patterns to lift: `actions/cache@v4` for VS integration, `conda-incubator/setup-miniconda@v3.1.0` for mamba, `Invoke-RestMethod` for the CUDA installer zip (Phase 2's concern, not Phase 1).
- **`ilammy/msvc-dev-cmd@v1`**: External GitHub Action for activating MSVC toolsets. Accepts `toolset:` and `vs-version:` inputs. Locked by TC-03.
- **`vswhere`**: Ships with Visual Studio Installer on `windows-2022`; available at a well-known path. Used here to probe for `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` before attempting install.
- **`vs_installer.exe modify`**: Official Microsoft mechanism for side-by-side toolset install on an existing VS install. Locked by TC-02.
- **Reference caching pattern**: `C:\claude_checkouts\keepassxc\.github\workflows\build.yml` (noted in PROJECT.md). Not relevant to Phase 1 (no caches here), but relevant to Phase 2.

### Established Patterns

- Upstream uses PowerShell (`shell: pwsh`) throughout `build-wheels-cuda.yaml`. Phase 1 follows the same convention for all Windows steps; the Ubuntu `lint-workflow` job uses bash.
- Upstream pins external actions with `@v4` or `@v5` semantic tags (e.g., `actions/checkout@v4`, `actions/setup-python@v5`). Phase 1 follows suit.
- Upstream uses `runs-on: ${{ matrix.os }}`; Phase 1 uses `runs-on: windows-2022` explicitly (TC-01, no matrix in v1).
- The repo has no existing ruff / lint configuration for YAML workflows — the `lint-workflow` job is a new lightweight CI surface, not layered on top of anything existing.

### Integration Points

- `vendor/llama.cpp/` submodule — preflight asserts `vendor/llama.cpp/CMakeLists.txt` exists after `submodules: recursive` checkout (WF-05). Phase 2 will read the submodule SHA from `vendor/llama.cpp/.git/HEAD` for wheel versioning (BLD-12) — Phase 1's forensics summary already includes this SHA, so Phase 2 will reuse the same resolution logic.
- `.github/workflows/build-wheels-cuda.yaml` (upstream) — Phase 1's YAML lives at a DIFFERENT path (`build-wheels-cuda-windows.yaml`) so upstream rebases don't stomp it. Neither file references the other.
- `llm_for_ci_test/` directory — Phase 3 territory (smoke-test fixture). Phase 1 does not touch it.

</code_context>

<deferred>
## Deferred Ideas

- **Repository-wide ban-grep lint as always-on PR check** — suggested during discussion as broader coverage than the phase-1 workflow-file-only grep. Out of scope for Phase 1 (phase boundary is the scaffold/toolchain YAML). Note for v2 / maintenance phase (`AUT-*`): a `.github/workflows/lint-workflows.yaml` that greps every CI file we own on every PR would be a sensible automation addition.
- **Composite action for preflight assertions** — would help if Phase 3's smoke-test ends up reusing some of the assertion logic. PROJECT.md constraint ("no shell scripts outside the workflow, keep debugging surface small") currently argues against this, but if Phase 3 ends up duplicating ≥3 assertions, revisit.
- **Chocolatey `vs-buildtools` as alternate toolset source** — research SUMMARY flagged it as a fallback. Rejected for Phase 1 (silent ABI drift risk). Keep in mind as an emergency lever if GitHub's VS 17.9 image ever becomes truly unavailable — would be a new phase in the v2 maintenance track.
- **Auto-downgrade to nearest installed 14.x** — rejected (same class of error as `-allow-unsupported-compiler`). Do not reconsider without a `_MSC_VER`-cap-aware variant and evidence from the smoke-test gate.
- **YAML workflow lint as a separate always-on job** (`actionlint`, `yamllint`) — mentioned in passing, not discussed. Worth considering alongside the ban-grep check in a future dedicated lint workflow, but not Phase 1.

</deferred>

---

*Phase: 01-scaffold-toolchain-pinning*
*Context gathered: 2026-04-15*
