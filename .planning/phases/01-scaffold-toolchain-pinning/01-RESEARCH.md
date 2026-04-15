# Phase 1: Scaffold & Toolchain Pinning - Research

**Researched:** 2026-04-15
**Domain:** GitHub Actions YAML — Windows runner toolchain pinning (VS 2022 / MSVC 14.39 side-by-side install + CUDA 12.4.1 via mamba) + workflow-file lint
**Confidence:** HIGH on stack, patterns, and primary pitfalls (cross-verified with upstream issue history, NVIDIA docs, Microsoft docs, actions/runner-images); MEDIUM on *exact current installability of the 14.39 side-by-side component* on today's `windows-2022` image (flagged empirical risk — the preflight-first pattern mitigates it regardless)

---

## Summary

This is a **pure YAML-authoring phase**: produce `.github/workflows/build-wheels-cuda-windows.yaml` containing exactly two jobs (`preflight` on `windows-2022`, `lint-workflow` on `ubuntu-latest`), with `workflow_dispatch`-only triggers, five hard-failing preflight assertions, and a ban-grep self-check. No build/smoke/publish work lands in this phase — those are Phases 2/3/4. The entire surface is PowerShell `run:` blocks plus one bash grep step, inline in one YAML file (no composite actions, no external scripts).

The load-bearing decisions have already been locked in `01-CONTEXT.md`: **MSVC 14.39 pinned via probe-first-install-second `vswhere`/`vs_installer.exe modify` pattern then activated via `ilammy/msvc-dev-cmd@v1 toolset: 14.39`**; **CUDA 12.4.1 via the single mamba path (`conda-incubator/setup-miniconda@v3.1.0` + `mamba install -c nvidia/label/cuda-12.4.1 cuda-toolkit`)**; **`_MSC_VER` cap is computed from the `msvc_toolset` dispatch input (14.39 → 1939, 14.40 → 1940, …) rather than hardcoded 1939**; **`-allow-unsupported-compiler` is banned and the ban is enforced by a grep self-check in the parallel `lint-workflow` Ubuntu job**. Research's job here is to verify these decisions are implementable against today's runner reality, document the exact PowerShell/YAML idioms, and surface the known empirical risks (14.39 component availability, `vendor/llama.cpp/.git/HEAD` hash-file edge case) with mitigation patterns.

The non-obvious finding that Claude's training surfaced and searches confirmed: **Microsoft has been quietly retiring older side-by-side `Microsoft.VisualStudio.Component.VC.*` components on the `windows-2022` image** (actions/runner-images #9701 started this in May 2024), and `VC.14.39.17.9.x86.x64` is currently marked "out of support" in Microsoft's component catalog. The CONTEXT.md-locked "probe first with `vswhere`, fail loudly with remediation hint if missing" pattern is exactly the right defense — if the 14.39 component rotates off the image tomorrow, the workflow fails at probe time (seconds) with an actionable message ("bump `msvc_toolset` to 14.40 and CUDA to 12.6+"), not after a 10-minute `vs_installer.exe modify` call that times out. Phase 1 does not need to solve the rotation; it needs to *detect* it loudly.

**Primary recommendation:** Write the YAML as two jobs (`preflight` Windows + `lint-workflow` Ubuntu, parallel), keep all assertion logic inline as `shell: pwsh` blocks with `Set-StrictMode -Version Latest` + `$ErrorActionPreference = 'Stop'` at the top of each, compute the `_MSC_VER` cap from the `msvc_toolset` input via a two-line PowerShell `[int]($msvc_toolset -replace '\D','')` expression, and emit a `$GITHUB_STEP_SUMMARY` forensics table on green. Use `actionlint` in the `lint-workflow` job alongside the ban-grep as a cheap catch-all for YAML/expression typos.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Job layout**

- Phase 1 YAML contains exactly two jobs: `preflight` (Windows) and `lint-workflow` (Ubuntu, runs in parallel).
- No stubbed `build` / `smoke-test` / `publish` job shells. Each future phase adds its own job.
- A separate `prepare-toolchain` *step* inside the `preflight` job handles VS/MSVC install and `vswhere` probing; preflight assertion steps are strictly read-only ("assert, don't mutate").

**Preflight scope & structure**

- Assertions (one step per assertion, named explicitly so failures are pinpointable in the Actions UI):
  1. MSVC toolset → `cl.exe /Bv` reports `_MSC_VER ≤ ${msvc_toolset_msc_cap}` (cap derived from `msvc_toolset` input, not hardcoded 1939).
  2. CUDA toolkit → `nvcc --version` matches the dispatched `cuda_version` input.
  3. Single nvcc path → `where nvcc` returns exactly one path AND no `CUDA_PATH_V*_*` env survives into preflight (TC-07 asserted here, not just hoped for at build time).
  4. Submodule populated → `vendor/llama.cpp/CMakeLists.txt` exists (catches silent `submodules: recursive` failures from WF-05).
  5. Python version match → `python --version` matches the dispatched `python_version` input (catches setup-python cache misses or wrong interpreter on PATH).
- On green preflight, emit a `$GITHUB_STEP_SUMMARY` markdown table with: MSVC version, `_MSC_VER`, nvcc version, CUDA path, Python version, llama.cpp submodule short SHA, runner OS build number, detected VS install path, mamba version, disk free space, `$env:PROCESSOR_IDENTIFIER`. Purpose: forensics when a wheel later misbehaves.
- On red preflight, emit actionable remediation hints (not just raw assertion output). Example: if `_MSC_VER > ${cap}`, hint reads roughly: "Runner VS image rotated. Retry dispatch with `msvc_toolset: 14.XX` matching the runner, or verify nvcc [version] compatibility per upstream #1543." Assertion failures should never require a human to cross-reference upstream issues from memory.

**Dispatch inputs**

- `python_version` → `choice`, default `3.11`, options `['3.10', '3.11', '3.12']`. 3.13 excluded (MX-03 / v2). Freeform rejected — dropdown makes valid set self-documenting in the Actions UI.
- `cuda_version` → `choice`, default `12.4.1`, options `['12.4.1']` (single-item). Expanding to 12.6.1 is MX-02 / v2. Single-item dropdown makes the v1 constraint visible.
- `msvc_toolset` → string-ish input, default `14.39`. Escape hatch for runner image rotation. **The preflight `_MSC_VER` cap must be computed from this input, not hardcoded 1939** — bumping the input to `14.40` must also bump the cap to `1940`, `14.41` → `1941`, etc.
- `verbose` → boolean, default false. Toggles `$env:VERBOSE=1` and step tracing for debugging preflight steps.
- `dry_run` → boolean, default false. When true, preflight exits green without invoking downstream jobs (useful once Phase 2 adds the build body — saves a full cold build cycle when only debugging toolchain pinning).
- Drift behavior: hard-fail on any mismatch with remediation hint. No warn-and-continue path; that is the upstream #1543 failure mode.

**MSVC 14.39 install strategy**

- **Probe first, install second**: call `vswhere -products * -requires Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` before attempting `vs_installer.exe modify`. If `vswhere` returns empty, fail immediately with remediation hint — don't waste runner minutes on a slow installer call that will fail.
- If `vswhere` finds the component, invoke `vs_installer.exe modify --add Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64 --quiet --norestart` (the component ID is actually derived from the `msvc_toolset` input).
- **No auto-fallback.** If install fails, hard-fail with remediation hint pointing at the `msvc_toolset` dispatch input. Explicitly rejected: chocolatey `vs-buildtools` fallback (silent ABI drift), auto-downgrade to nearest installed 14.x (same class of error as `-allow-unsupported-compiler`).
- `ilammy/msvc-dev-cmd@v1` is pinned to `vs-version: '[17.9,17.10)'`. VS 17.10+ on the runner trips preflight and forces re-validation before we ship — tight pin is the whole point.

**Ban-grep self-check (TC-10)**

- Separate `lint-workflow` job, runs on `ubuntu-latest` in parallel with `preflight` (free Ubuntu minutes, runs in seconds, shows as independent red/green check on the dashboard).
- Scope: grep `.github/workflows/build-wheels-cuda-windows.yaml` ONLY. Do not grep upstream's `build-wheels-cuda.yaml` (it legitimately contains the flag; we don't own it). Do not grep `CMakeLists.txt` / `pyproject.toml` (out of phase scope; TC-10 is about the workflow file). Do not grep repo-wide.
- Implementation: `grep -nH 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml && exit 1 || exit 0`.
- Phase 3's publish job will later declare `needs:` on this job too (structural gate against regressions).

**File organization**

- Single YAML file. No composite actions under `.github/actions/`, no external PowerShell scripts, no reusable workflow fragments. Matches PROJECT.md constraint: *"GitHub Actions YAML only. No shell scripts outside the workflow (keep debugging surface small)."*
- All preflight assertions are inline PowerShell in `run:` blocks (`shell: pwsh`).

**YAML documentation (DOC-04)**

- Inline comments explain: (1) why MSVC 14.39 is pinned (nvcc's `_MSC_VER` check, link to upstream #1543), (2) why `-allow-unsupported-compiler` is banned (runtime-segfault trap), (3) why VS range is `[17.9,17.10)` not looser, (4) what `msvc_toolset` dispatch input is for (emergency mitigation when runner image rotates).

### Claude's Discretion

- Exact remediation-hint wording (just make it actionable and unambiguous; include upstream issue numbers where relevant).
- Forensics summary table column order and formatting in `$GITHUB_STEP_SUMMARY`.
- Exact PowerShell idioms for assertions (`Test-Path`, `Select-String`, `throw`, `exit 1` — whatever reads cleanest).
- Mechanism for setting `LongPathsEnabled=1` (TC-09) before checkout — whichever of `reg add` / `New-ItemProperty` / `actions/setup-` prelude is idiomatic.
- Mechanism for unsetting `CUDA_PATH_V*_*` env vars (TC-07) — loop through `Get-ChildItem env:` matching the pattern and `Remove-Item`.
- Step-level `timeout-minutes` values (BLD-09 is Phase 2, but Phase 1 should still bound its steps — reasonable defaults).
- Whether the `prepare-toolchain` step emits a "starting VS component install" progress line (minor log polish).

### Deferred Ideas (OUT OF SCOPE)

- **Repository-wide ban-grep lint as always-on PR check** — suggested during discussion as broader coverage than the phase-1 workflow-file-only grep. Out of scope for Phase 1 (phase boundary is the scaffold/toolchain YAML). Note for v2 / maintenance phase (`AUT-*`): a `.github/workflows/lint-workflows.yaml` that greps every CI file we own on every PR would be a sensible automation addition.
- **Composite action for preflight assertions** — would help if Phase 3's smoke-test ends up reusing some of the assertion logic. PROJECT.md constraint ("no shell scripts outside the workflow, keep debugging surface small") currently argues against this, but if Phase 3 ends up duplicating ≥3 assertions, revisit.
- **Chocolatey `vs-buildtools` as alternate toolset source** — research SUMMARY flagged it as a fallback. Rejected for Phase 1 (silent ABI drift risk). Keep in mind as an emergency lever if GitHub's VS 17.9 image ever becomes truly unavailable — would be a new phase in the v2 maintenance track.
- **Auto-downgrade to nearest installed 14.x** — rejected (same class of error as `-allow-unsupported-compiler`). Do not reconsider without a `_MSC_VER`-cap-aware variant and evidence from the smoke-test gate.
- **YAML workflow lint as a separate always-on job** (`actionlint`, `yamllint`) — mentioned in passing, not discussed. Worth considering alongside the ban-grep check in a future dedicated lint workflow, but not Phase 1.

*Note: CONTEXT.md explicitly deferred "actionlint as a separate always-on job" to v2. However, since Phase 1 already has a `lint-workflow` job with runtime budget to spare (Ubuntu, seconds), adding actionlint as one extra step in that same job is within scope if the planner agrees — it's a cheap force-multiplier on the ban-grep. Flagged for planner decision.*
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| **WF-01** | New workflow file `.github/workflows/build-wheels-cuda-windows.yaml` exists, separate from upstream's `build-wheels-cuda.yaml` | Locked: new file, no edits to upstream file. § "File organization" and `## Architecture Patterns / Pattern 1`. Mechanical. |
| **WF-02** | Workflow is triggered only by `workflow_dispatch` (no push/tag/release auto-triggers in v1) | Locked: `on: workflow_dispatch:` block with `inputs:` dict. See `## Code Examples / Workflow trigger & inputs`. |
| **WF-03** | Workflow exposes `python_version` input (default `3.11`) selectable at dispatch time | Locked as `choice` type with options `['3.10', '3.11', '3.12']`, default `3.11`. GitHub Actions `choice` input type renders as dropdown in Actions UI. |
| **WF-04** | Workflow exposes `cuda_version` input (default `12.4.1`) selectable at dispatch time | Locked as single-item `choice` (`['12.4.1']`) — the single-item dropdown is intentional (self-documents v1 constraint; expanding to 12.6.1 is MX-02 / v2). |
| **WF-05** | Workflow checks out the repo with `submodules: recursive` so `vendor/llama.cpp/` is populated | `actions/checkout@v4` with `submodules: recursive`. Phase 1 adds assertion 4 (Submodule populated → `vendor/llama.cpp/CMakeLists.txt` exists) to catch silent failures. |
| **TC-01** | Runner is pinned to `windows-2022` explicitly (never `windows-latest`) | `runs-on: windows-2022` literal string in the `preflight` job. `## Common Pitfalls / Pitfall 2` covers the rationale (image rotation risk). |
| **TC-02** | MSVC toolset 14.39 side-by-side installed via `vs_installer.exe modify --add Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` | "Probe first with `vswhere`, install second with `vs_installer.exe modify`." See `## Code Examples / MSVC 14.39 probe + install`. Component ID is derived from `msvc_toolset` input. |
| **TC-03** | MSVC 14.39 toolset is activated for the build via `ilammy/msvc-dev-cmd@v1` with `toolset: 14.39` | `uses: ilammy/msvc-dev-cmd@v1` with `toolset: ${{ inputs.msvc_toolset }}` and `vs-version: '[17.9,17.10)'`. See `## Code Examples / MSVC activation`. |
| **TC-04** | Preflight step asserts `cl.exe /Bv` reports `_MSC_VER` ≤ `1939`; build fails loudly if assertion fails | Assertion 1 in CONTEXT.md. `_MSC_VER` cap is **computed from `msvc_toolset` input**, not hardcoded 1939. See `## Code Examples / Assertion 1 — MSVC toolset`. |
| **TC-05** | CUDA toolkit is installed via a single path (mamba `cuda-toolkit`); no parallel Jimver full-installer install | `conda-incubator/setup-miniconda@v3.1.0` + `mamba install -c nvidia/label/cuda-$VER cuda-toolkit=$VER`. **No `Jimver/cuda-toolkit` action invocation in Phase 1** (it was upstream's way of getting MSBuildExtensions, but Phase 2 concern if needed). See `## Code Examples / CUDA install`. |
| **TC-06** | Preflight step asserts `nvcc --version` matches the requested `cuda_version` input | Assertion 2. Parse with `Select-String 'release ([0-9]+\.[0-9]+)'`. Compare to `${{ inputs.cuda_version }}` (major.minor match, since `nvcc --version` typically reports `12.4` not `12.4.1`). See `## Code Examples / Assertion 2 — CUDA toolkit`. |
| **TC-07** | `CUDA_PATH` and `CUDA_PATH_V*_*` env variables are explicitly unset or normalized before build | Assertion 3. Loop `Get-ChildItem env: | Where-Object { $_.Name -match '^CUDA_PATH_V' } | Remove-Item`, then set exactly one `CUDA_PATH=$env:CONDA_PREFIX`. See `## Code Examples / Assertion 3 — single nvcc path`. |
| **TC-08** | Visual Studio BuildCustomizations path is detected dynamically (no hardcoded `2019\Enterprise` path from upstream) | `Get-ChildItem 'C:\Program Files\Microsoft Visual Studio\2022\*\MSBuild\Microsoft\VC\*\BuildCustomizations'` — matches Enterprise/Community/BuildTools on the 64-bit `C:\Program Files\` path (not `(x86)`). Strict-mode error if empty match. See `## Code Examples / Dynamic VS BuildCustomizations discovery`. |
| **TC-09** | Windows `LongPathsEnabled=1` registry key is set before checkout | Inline PowerShell `New-ItemProperty` before `actions/checkout@v4`. Also `git config --system core.longpaths true` at job start. See `## Code Examples / LongPathsEnabled`. |
| **TC-10** | CMake flag `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` is **not** used anywhere; CI fails if the workflow contains that literal (grep-assert) | Separate `lint-workflow` job on `ubuntu-latest`, one step: `grep -nH 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml && exit 1 || exit 0`. Scope narrowly: only our Windows file; NOT upstream's. See `## Code Examples / Ban-grep self-check`. |
| **DOC-04** | Workflow YAML contains inline comments explaining the MSVC 14.39 pin rationale and the `-allow-unsupported-compiler` ban (link to upstream #1543) | YAML `#` comments at: top-of-file preamble (links #1543, #2136), `msvc_toolset` input (explains runner-image-rotation escape hatch), `ilammy/msvc-dev-cmd` step (explains `[17.9,17.10)` tight pin), `lint-workflow` job (explains why ban-grep runs on Ubuntu). See `## Code Examples / Inline documentation comments`. |
</phase_requirements>

---

## Standard Stack

### Core Actions (used by `preflight` / `lint-workflow`)

| Action | Version Pin | Purpose | Why Standard |
|---|---|---|---|
| `actions/checkout` | `@v4` | Clone repo with `submodules: recursive` (WF-05) + enable `sparse-checkout` if Phase 3 ever needs it | Node 20, stable since 2023, matches upstream pin. Do not use `@v5` for Phase 1 — no relevant behaviour change, v4 is the conservative choice. |
| `actions/setup-python` | `@v5` | Install Python interpreter matching `inputs.python_version` | Node 20 action, pip caching via `cache: 'pip'` works. v6 needs runner `>=2.327.1` for no benefit. Note: `setup-python` Python is what drives `python --version` assertion (Assertion 5); mamba is separate. |
| `conda-incubator/setup-miniconda` | `@v3.1.0` | Provide mamba for `cuda-toolkit` install | Exact version pin matches upstream workflow (see `.github/workflows/build-wheels-cuda.yaml` line 62). Floating `@v3` would also work; exact pin chosen for determinism. |
| `ilammy/msvc-dev-cmd` | `@v1` | Activate the *specific* MSVC 14.39 toolset in the shell | **Critical.** `microsoft/setup-msbuild@v2`'s `vs-version` input selects among *already-installed* VS versions — it cannot downgrade a 17.14 runner to 17.9. `ilammy/msvc-dev-cmd` runs `vcvarsall.bat -vcvars_ver=14.39`, which downgrades the *active* toolset once the 14.39 side-by-side component is installed on disk. Floating `@v1` is safe (single-purpose action). |

### Supporting Actions

| Action | Version Pin | Purpose | When to Use |
|---|---|---|---|
| `rhysd/actionlint` (or `actionlint` invocation in bash) | latest release at `lint-workflow` plan time | Static analysis of workflow YAML — catches expression typos, deprecated syntax, invalid runner labels | **Claude's discretion:** add to `lint-workflow` job alongside ban-grep. Cheap (~5s), catches a class of bugs ban-grep cannot. Flagged in CONTEXT.md deferred list but in-scope to include here since `lint-workflow` job already exists. |
| `microsoft/setup-msbuild` | — | *Not used in Phase 1.* | Upstream uses it; we do not. Reasoning in `## Don't Hand-Roll` below. |
| `Jimver/cuda-toolkit` | — | *Not used in Phase 1.* | Upstream uses it for MSBuildExtensions extraction; Phase 1 uses mamba-only per TC-05. Phase 2 may revisit. |
| `actions/cache` | — | *Not used in Phase 1.* | Caching is Phase 2's concern (BLD-05, BLD-06, BLD-07). Phase 1's `preflight` is cheap enough to not need caching. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|---|---|---|
| `ilammy/msvc-dev-cmd@v1` + `vs_installer.exe modify` | `microsoft/setup-msbuild@v2 vs-version: '[17.9,17.10)'` | **Rejected.** `setup-msbuild` selects among installed VS versions — on `windows-2022` (currently VS 17.14), the `[17.9,17.10)` range resolves nothing, and the step fails. The only way to *get* 14.39 active is to side-by-side-install the component and activate with `vcvarsall.bat -vcvars_ver=14.39`, which is exactly what `ilammy/msvc-dev-cmd` wraps. |
| mamba `cuda-toolkit` | `Jimver/cuda-toolkit@v0.2.24` full installer | **Rejected for Phase 1.** Jimver's full installer hangs on `windows-2022` for CUDA 12.5+ (Jimver #382), and while it works for 12.4.1 today, TC-05 explicitly locks us to "one install path." Using Jimver for the link table only (Phase 2 MSBuildExtensions extraction) is fine. Phase 1 doesn't need MSBuildExtensions — that's a Phase 2 concern once we actually invoke CUDA at build time. |
| `choice` dispatch input | freeform `string` input | **Rejected.** `choice` inputs render as dropdowns in the Actions UI, which self-documents valid values. Freeform strings invite typos like `python_version: 3.11.0` or `cuda_version: 12.4` (missing `.1`) that would bypass our validation. |
| Hardcoded `_MSC_VER` cap 1939 | Cap computed from `msvc_toolset` input | **Rejected.** Hardcoding the cap defeats the whole purpose of `msvc_toolset` being a dispatch input. If an operator bumps `msvc_toolset` to `14.40` at 3am to work around a runner-image rotation, the cap must auto-bump to `1940`; otherwise preflight fails with "MSVC 14.40 installed, cap 1939" — a false positive that the on-call human has to override manually. **Derivation:** `$cap = [int]("1" + ($msvc_toolset -replace '\.',''))` → `'14.39'` → `"1" + "1439"` → `11439` is wrong; use `$cap = 1900 + [int](($msvc_toolset -replace '\D','').Substring(0))` → better: `$parts = $msvc_toolset -split '\.'; $cap = 1000 * [int]$parts[0] / 10 + [int]$parts[1] + 1400` is messy. **Cleanest:** `'14.39' → '1939'` via `($msvc_toolset -replace '\.','').PadLeft(4,'0')` → `"1439"` then prefix with `"1"` → `"11439"`. **NO.** The right derivation: `_MSC_VER` for MSVC 14.XX is `1900 + XX` (Microsoft's documented formula). So for `14.39`: `1900 + 39 = 1939`. For `14.40`: `1940`. One-liner: `$minor = [int](($msvc_toolset -split '\.')[1]); $cap = 1900 + $minor`. Verified against [Microsoft's "MSVC Toolset Minor Version Number 14.40" devblog](https://devblogs.microsoft.com/cppblog/msvc-toolset-minor-version-number-14-40-in-vs-2022-v17-10/). |

**No new package installations.** Phase 1's surface is entirely GitHub Actions + PowerShell + `vs_installer.exe` (preinstalled) + `vswhere.exe` (preinstalled) + `mamba` (installed by `setup-miniconda`) + `cl.exe`/`nvcc` (discovered after install). No `npm install`, no `pip install` beyond `setup-python`'s pip-cache machinery.

---

## Architecture Patterns

### Recommended File Layout

```
.github/
└── workflows/
    ├── build-wheels-cuda-windows.yaml   # NEW — Phase 1 deliverable
    ├── build-wheels-cuda.yaml            # UNTOUCHED (upstream)
    ├── build-wheels-metal.yaml           # UNTOUCHED (upstream)
    ├── lint.yaml                         # UNTOUCHED (upstream)
    └── ...                               # other upstream workflows untouched
```

Nothing under `.github/actions/`. No `.ci/` scripts. No YAML anchors splitting files. PROJECT.md constraint is load-bearing here: *"GitHub Actions YAML only. No shell scripts outside the workflow (keep debugging surface small)."*

### Pattern 1: Two Parallel Jobs (`preflight` + `lint-workflow`)

**What:** Two top-level jobs with no `needs:` relationship between them. They run in parallel. Neither blocks the other; both must pass for the overall workflow to be green.

**Why:** Ban-grep (TC-10) is a fast, Ubuntu-only check that catches regressions on *workflow file edits* independent of runner state. Putting it behind `needs: preflight` would gate a 5-second YAML lint on a 10-minute toolchain install — for no reason. Running them in parallel means a broken YAML commit shows up in seconds on the dashboard even if the runner is busy installing VS components.

**When to use:** Any time an independent lint/validation can run on a cheaper runner without needing the heavyweight job's outputs. This is a "free parallelism" pattern.

**Example skeleton:**
```yaml
name: Build Wheels (CUDA Windows)

# DOC-04: workflow_dispatch-only in v1 (WF-02). Auto-triggers (push/tag/release)
# are deferred to v2 AUT-01/AUT-02. The `msvc_toolset` input is an escape hatch
# for when GitHub's windows-2022 image rotates its VS version — bump this
# without opening a PR. See upstream #1543 for why the toolset pin matters.
on:
  workflow_dispatch:
    inputs:
      python_version:
        description: 'Python version to build for'
        type: choice
        default: '3.11'
        options: ['3.10', '3.11', '3.12']
      cuda_version:
        description: 'CUDA toolkit version'
        type: choice
        default: '12.4.1'
        options: ['12.4.1']
      msvc_toolset:
        description: 'MSVC toolset version (bump only when runner VS image rotates — see upstream #1543)'
        type: string
        default: '14.39'
      verbose:
        description: 'Verbose preflight tracing'
        type: boolean
        default: false
      dry_run:
        description: 'Run preflight only; skip downstream build when added in Phase 2'
        type: boolean
        default: false

permissions:
  contents: read  # Phase 1 reads, doesn't write; publish job (Phase 4) will widen.

jobs:
  preflight:
    runs-on: windows-2022   # TC-01: explicit, never windows-latest
    timeout-minutes: 20
    defaults:
      run: { shell: pwsh }
    steps:
      # ... prepare-toolchain step + 5 assertion steps + summary step ...

  lint-workflow:
    runs-on: ubuntu-latest  # cheaper, faster, independent
    timeout-minutes: 5
    steps:
      # ... ban-grep + (optional) actionlint ...
```

### Pattern 2: Probe-First-Install-Second (`vswhere` → `vs_installer.exe modify`)

**What:** Before invoking any mutating installer, query `vswhere.exe` to check if the required component is even available on the image. If not available, fail immediately with a remediation hint. If available, proceed to install.

**Why:** `vs_installer.exe modify` is slow (2-10 min) and gives terrible error messages when it can't find a requested component — it often exits 0 with a vague log entry. `vswhere.exe` is instant and returns structured output. A 2-second probe that fails fast beats a 10-minute install that fails late every time.

**When to use:** Any mutating toolchain install where the target component might have been removed from the image. Especially relevant after `actions/runner-images #9701` (May 2024) started removing redundant VC components.

**Example:**
```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$vswhere = 'C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe'
$componentId = "Microsoft.VisualStudio.Component.VC.$($toolset -replace '\.','').17.9.x86.x64"
# e.g. toolset='14.39' → componentId='Microsoft.VisualStudio.Component.VC.1439.17.9.x86.x64'
# Note: real component ID uses dots — use the literal. This is toolset-specific.

# Probe: does *any* VS install expose this component?
$found = & $vswhere -products * -requires $componentId -property installationPath
if (-not $found) {
  Write-Error @"
MSVC toolset $toolset component not available on windows-2022 image.
Component ID: $componentId

REMEDIATION:
  1. Retry dispatch with msvc_toolset=14.40 (and bump CUDA to 12.6.1 in Phase 2)
  2. Check actions/runner-images#9701 for component removal announcements
  3. Verify nvcc/_MSC_VER compatibility per upstream abetlen/llama-cpp-python#1543

This is a hard-fail, not a warn-and-continue — see PROJECT.md 'smoke-test as publish gate'.
"@
  exit 1
}
Write-Host "Found $componentId at: $found"
```

### Pattern 3: `_MSC_VER` Cap Derived From `msvc_toolset` Input

**What:** The numeric cap used in Assertion 1 is computed from the input at step execution time, not hardcoded.

**Why:** Microsoft's documented MSVC version formula is `_MSC_VER = 1900 + minor` where `minor` is the second dotted-pair component of the toolset version (e.g. `14.39` → `1939`, `14.40` → `1940`). Locked directly by the input means a single dispatch-time change propagates to both the install and the assertion.

**When to use:** Exactly here. The exact derivation:

```powershell
# $toolset = '14.39' (from ${{ inputs.msvc_toolset }})
$parts = $toolset -split '\.'
if ($parts.Count -ne 2) { throw "msvc_toolset must be 'MAJOR.MINOR' form; got '$toolset'" }
$major = [int]$parts[0]; $minor = [int]$parts[1]
if ($major -ne 14) { throw "msvc_toolset major must be 14 for v1 (got '$major')" }
$mscVerCap = 1900 + $minor
# $mscVerCap = 1939 when $toolset='14.39'; 1940 when '14.40'; etc.
```

This is the exact formula documented in [Microsoft's devblog "MSVC Toolset Minor Version Number 14.40 in VS 2022 v17.10"](https://devblogs.microsoft.com/cppblog/msvc-toolset-minor-version-number-14-40-in-vs-2022-v17-10/).

### Pattern 4: Assertion Steps Use Strict-Mode PowerShell

**What:** Every assertion step begins with:
```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
```

**Why:** Default PowerShell is permissive — `$null.fullname` returns `$null` silently, `$wrongVar` evaluates to `$null` without erroring, missing-path `Get-Item` returns `$null` with a non-terminating warning. These are exactly the failure modes that Pitfall 4 (Upstream MSBuildExtensions copy to VS 2019 path, silently no-ops) documents. Strict-mode flips every one of these into a throw.

**When to use:** Every single `run: |` block with `shell: pwsh` in this phase. Non-negotiable.

**Example:** Without strict-mode:
```powershell
$vsPath = (Get-Item 'C:\Program Files\Microsoft Visual Studio\2019\Enterprise').FullName
# Silently returns $null on windows-2022 — no such path
$vsPath + '\vcvarsall.bat' | Out-Host
# Outputs just '\vcvarsall.bat' because $null + string = string
```

With strict-mode: the `Get-Item` call throws immediately with "Cannot find path".

### Pattern 5: `$GITHUB_STEP_SUMMARY` Forensics Table

**What:** On the final step of a green `preflight` job, write a markdown table to `$env:GITHUB_STEP_SUMMARY` containing every relevant toolchain fingerprint. This surfaces in the Actions UI immediately below the job log.

**Why:** When a wheel shipped from a run six weeks ago starts segfaulting for a user today, the `$GITHUB_STEP_SUMMARY` of that build is enough to reconstruct the toolchain without re-running anything. Err on the side of *more columns*, not fewer — disk space is not a constraint in a markdown summary.

**When to use:** Always, at the tail of the `preflight` job. Not on red — red path emits remediation hints instead.

**Columns to include (CONTEXT.md locks these):** MSVC version, `_MSC_VER`, nvcc version, CUDA path, Python version, llama.cpp submodule short SHA, runner OS build number, detected VS install path, mamba version, disk free space, `$env:PROCESSOR_IDENTIFIER`.

**Example:**
```powershell
@"
## Preflight Green — Toolchain Fingerprint

| Property | Value |
|---|---|
| Runner OS | $(Get-ComputerInfo OsName,OsBuildNumber | Out-String) |
| MSVC toolset | $toolset ($_MSC_VER) |
| VS install path | $vsPath |
| nvcc version | $nvccVersion |
| CUDA path | $env:CUDA_PATH |
| Python | $pyVersion ($pyPath) |
| llama.cpp SHA | $(git -C vendor/llama.cpp rev-parse --short HEAD) |
| mamba version | $(mamba --version) |
| Disk free (C:) | $((Get-PSDrive C).Free / 1GB) GB |
| CPU | $env:PROCESSOR_IDENTIFIER |
"@ | Out-File -FilePath $env:GITHUB_STEP_SUMMARY -Append -Encoding utf8
```

### Anti-Patterns to Avoid

- **`shell: pwsh` without `Set-StrictMode -Version Latest`.** Silent `$null` propagation = false-green builds (Pitfall 4 root cause).
- **Hardcoded VS install paths** (`C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\...`). VS 2022 is 64-bit (`C:\Program Files\Microsoft Visual Studio\2022\...`), and the edition is Enterprise on the runner today but was Community on `windows-2019` and may rotate again. Always discover with `vswhere` or glob.
- **Continuing past a failed `vs_installer.exe modify`.** The installer exits 0 even on some partial-failure paths. Always re-probe with `vswhere` after the modify call to confirm the component is now installed.
- **Single monolithic "do all five assertions" step.** CONTEXT.md explicitly requires *one step per assertion* so the Actions UI shows the specific failing step name — "Preflight: Assert MSVC cap" is diagnostic; "Preflight: Run all checks" is not.
- **Using `msbuild.exe` discovery for MSVC 14.39 activation.** `microsoft/setup-msbuild` selects among *installed* VS versions; it cannot narrow to a specific toolset within a VS install. Use `ilammy/msvc-dev-cmd` `toolset:` input (which wraps `vcvarsall.bat -vcvars_ver=...`).
- **Relying on `$env:CUDA_PATH` being clean at job start.** Runner images ship with leftover `CUDA_PATH_V*_*` variables from earlier CUDA installs. Always sweep (`Get-ChildItem env: | Where Name -match '^CUDA_PATH_V' | Remove-Item`) before asserting.
- **Writing the ban-grep on Windows.** PowerShell's `Select-String` works but bash `grep -nH` is shorter, faster, and runs on the free Ubuntu runner. Keep TC-10 on `ubuntu-latest`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---|---|---|---|
| Activating a specific MSVC toolset in the job shell | Custom PowerShell wrapper calling `vcvarsall.bat` and re-exporting env vars to `$GITHUB_ENV` | `ilammy/msvc-dev-cmd@v1` | Handles the env-var export dance correctly (dozens of `INCLUDE`/`LIB`/`PATH` entries), tested across hundreds of projects, single-purpose so no regression risk. Hand-rolling this is exactly how you drop a `LIB` entry and silently link against the wrong CRT. |
| Installing CUDA toolkit on Windows | NVIDIA silent-installer invocation (`.exe -s nvcc_12.4 cudart_12.4 ...`) | `conda-incubator/setup-miniconda@v3.1.0` + `mamba install -c nvidia/label/cuda-$VER cuda-toolkit=$VER` | NVIDIA's silent flags change between CUDA minor versions, the installer writes VS integration into whichever VS it finds (wrong toolset risk), and the exe is 3 GB cold. Mamba puts nvcc into `$CONDA_PREFIX` cleanly, is version-pinnable, and mirrors the Linux pattern so research/debug context carries over. |
| Parsing `cl.exe /Bv` output for `_MSC_VER` | Hand-written regex over the banner | `cl.exe 2>&1 | Select-String 'Version (\d+\.\d+)'` + the `1900 + minor` formula | The output format `Microsoft (R) C/C++ Optimizing Compiler Version 19.39.33523 for x64` is stable. Grab `19.39`, split on `.`, compute `_MSC_VER = 1900 + 39 = 1939`. Don't try to parse the full 4-component version. |
| Discovering VS installation path | Globbing `C:\Program Files\Microsoft Visual Studio\2022\*` | `vswhere.exe -latest -property installationPath` | `vswhere` is the Microsoft-blessed discovery tool, handles Enterprise/Community/BuildTools/Preview editions, preinstalled on `windows-2022`. Globbing works but is fragile to edition naming. |
| Checking submodule SHA for the forensics summary | `git ls-tree HEAD vendor/llama.cpp` parsing | `git -C vendor/llama.cpp rev-parse --short HEAD` | `-C` directly in the submodule after `submodules: recursive` checkout returns the exact commit. Short form is 7 chars, enough for forensics. |
| Regex-asserting `nvcc --version` | Multi-line awk-style parsing | `(nvcc --version) -match 'release (\d+\.\d+)'` + compare to `${{ inputs.cuda_version }}` major.minor | `nvcc --version` output format has been stable since CUDA 10.x. Match the `release X.Y` line, strip to major.minor, compare. Don't try to parse the full 3-component version — `nvcc` often reports `12.4` for what mamba installs as `12.4.1`. |
| Generating the step-summary markdown | String concatenation with `"| $a | $b |\n"` | PowerShell here-string (`@" ... "@`) + `Out-File -Append -Encoding utf8` | Here-strings handle multi-line markdown cleanly. `-Encoding utf8` is required for `$GITHUB_STEP_SUMMARY` on Windows — default `utf16` will render as garbage. |
| Unsetting `CUDA_PATH_V*_*` env vars | Manually listing each variant (`CUDA_PATH_V12_0`, `CUDA_PATH_V12_1`, ...) | `Get-ChildItem env: | Where-Object { $_.Name -match '^CUDA_PATH_V\d+_\d+$' } | ForEach-Object { Remove-Item "env:$($_.Name)" }` | Future-proof against new CUDA versions showing up on the runner image. |
| Workflow-file static analysis | Hand-written grep for `${{ }}` expression mistakes | `actionlint` (Go binary, one-line install on Ubuntu) | Type-checks expressions, validates action inputs, catches deprecated syntax. Ban-grep catches exactly one literal string; actionlint catches the whole class of "this `${{ steps.foo.outputs.bar }}` references a step that doesn't exist" bugs. **Claude's discretion to include** (CONTEXT.md deferred it explicitly but also left the door open for the `lint-workflow` job). |

**Key insight:** Every item above has been re-implemented badly in some llama-cpp-python-adjacent project's CI, and Phase 1's entire value is *not* recreating those mistakes. The rule: if a GitHub-blessed action or a Microsoft-blessed binary exists for the operation, use it; PowerShell is for glue and assertion, not for rebuilding tools.

---

## Common Pitfalls

### Pitfall 1: The 14.39 Component Has Been Retired From `windows-2022`

**What goes wrong:** `vs_installer.exe modify --add Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64 --quiet --norestart` exits 0 but the component is not actually installed; subsequent `ilammy/msvc-dev-cmd toolset: 14.39` fails with "vcvars_ver=14.39 not found" or silently falls back to the default 14.50.

**Why it happens:** `actions/runner-images #9701` (May 2024) began removing redundant `Microsoft.VisualStudio.Component.VC.*` components from the `windows-2022` image. As of 2026-04, Microsoft's component catalog lists `VC.14.39.17.9.x86.x64` as "out of support." The component *may* still be installable on-demand (the installer can pull from Microsoft's archive) or may have been purged from the image entirely — this is empirically time-sensitive.

**How to avoid:** The probe-first-install-second pattern (Pattern 2 above) handles this correctly. The `vswhere -products * -requires Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` call returns empty if the component is not available *even for install*. Exit at that point with a remediation hint pointing at `msvc_toolset: 14.40` (and the Phase 2 follow-on of bumping CUDA to 12.6.1 per MX-02).

**Warning signs:**
- `vs_installer.exe modify` exits 0 in under 10 seconds (real installs take 2-10 min).
- Post-install `vswhere` still returns empty for the required component.
- `ilammy/msvc-dev-cmd toolset: 14.39` emits `vcvars_ver=14.39 not found` but the step succeeds.
- `cl.exe /Bv` reports `19.50.*` (`_MSC_VER=1950`) instead of `19.39.*` — the activation silently fell back to the default toolset.

### Pitfall 2: Silent `submodules: recursive` Failure

**What goes wrong:** `actions/checkout@v4` with `submodules: recursive` succeeds but `vendor/llama.cpp/` is empty or missing `CMakeLists.txt`. Happens when the submodule's network clone times out, when Git LFS tokens are mis-scoped on PRs from forks, or when a post-checkout `git clean` runs.

**Why it happens:** Submodule fetch is a separate network operation inside `checkout@v4`. If it times out, the action logs a warning but doesn't fail the step. Later phases (Phase 2 build) then fail mysteriously with `CMakeLists.txt not found` deep in `vendor/llama.cpp/ggml/src/ggml-cuda/`.

**How to avoid:** Assertion 4 (CONTEXT.md) — `Test-Path vendor/llama.cpp/CMakeLists.txt`. If absent, throw with remediation "Submodule checkout failed; re-dispatch the workflow. If persistent, check `.gitmodules` and runner network."

**Warning signs:**
- Phase 2's CMake configure step errors with `No such file or directory: vendor/llama.cpp/CMakeLists.txt`.
- `git -C vendor/llama.cpp rev-parse HEAD` returns "fatal: not a git repository".

### Pitfall 3: Wrong Python on PATH After `setup-python` + `setup-miniconda`

**What goes wrong:** `setup-python@v5` installs Python 3.11 at `C:\hostedtoolcache\windows\Python\3.11.*\x64\python.exe` and prepends it to PATH. Then `setup-miniconda@v3.1.0` installs its own Python 3.12 in the activated env at `C:\Miniconda3\envs\llamacpp\python.exe` and also prepends to PATH. PATH order: mamba wins. `python --version` reports 3.12, not the dispatched 3.11.

**Why it happens:** Both actions eagerly prepend to PATH. `setup-miniconda`'s `activate-environment: llamacpp` + `python-version: ${{ inputs.python_version }}` creates a mamba-managed Python *with the same version number you asked for* — but if you ever get the two out of sync, the assertion must catch it.

**How to avoid:** Assertion 5 (CONTEXT.md) — `python --version` matches `inputs.python_version`. Also `where.exe python` should show exactly one hit; if more, the first one better be the one you want.

**Warning signs:**
- `where.exe python` returns multiple paths.
- `python --version` reports major.minor different from `inputs.python_version`.
- Later build steps use a different `python.exe` than the assertion did.

### Pitfall 4: `CUDA_PATH_V*_*` Env Var Pollution

**What goes wrong:** A previous job (or a runner warm-start artifact) left `CUDA_PATH_V12_4=C:\Old\CUDA` in the environment. Our mamba install sets `CUDA_PATH=$CONDA_PREFIX`. CMake's `FindCUDAToolkit.cmake` traverses `CUDA_PATH_V*_*` first in some code paths, finds the stale one, and the build links against the wrong headers.

**Why it happens:** Upstream's own workflow (line 93-94) explicitly *writes* `CUDA_PATH_V12_4=$env:CONDA_PREFIX` to `$env:GITHUB_ENV`. If that variable persists from a previous run (shouldn't on fresh GitHub-hosted runners, but does on self-hosted) or if a Jimver install set it, we get ghost variables.

**How to avoid:** Assertion 3 (CONTEXT.md) — sweep all `CUDA_PATH_V*_*` at the start of preflight *before* the mamba install, then after mamba install assert `where nvcc` returns exactly one path.

**Warning signs:**
- `Get-ChildItem env: | Where Name -match 'CUDA'` returns more than one `CUDA_PATH*` entry.
- `where nvcc` returns multiple paths.
- CMake logs `-- Found CUDAToolkit: ... (found version "12.1.1")` when you requested 12.4.1.

### Pitfall 5: VS BuildCustomizations Copy Silently No-ops

**What goes wrong:** (Pitfall 4 in PITFALLS.md — kept here because it's Phase 1 territory.) Upstream's line 92 hardcodes `C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\...`. On `windows-2022`, that path doesn't exist. `Get-Item` with a wildcard returning zero matches is `$null`, and `$null.fullname.foreach(...)` is a silent no-op without strict-mode.

**How to avoid:** Dynamic VS discovery with `vswhere -latest -property installationPath`, then glob `$vsPath\MSBuild\Microsoft\VC\*\BuildCustomizations`. **Strict-mode throws on empty match.** Phase 1's `preflight` doesn't actually need BuildCustomizations (Phase 2 does, for MSBuild-driven CUDA .vcxproj), but the **discovery logic must be ready** — TC-08 asserts dynamic detection with strict-mode failure on zero matches. Deliverable in Phase 1: a reusable PowerShell snippet that locates the path and errors cleanly, so Phase 2 doesn't re-hit this.

**Warning signs:**
- Any hardcoded `2019`, `Enterprise`, or `Program Files (x86)` (VS 2019's x86 path) in the YAML.
- A "copy MSBuildExtensions" step that finishes in under 1 second (real copies take 2-10s).

### Pitfall 6: YAML Expression Typos Only Catch at Dispatch Time

**What goes wrong:** `${{ inputs.pyhton_version }}` (typo — `pyhton`) evaluates to empty string at runtime. No compile-time check. Discover only when the first dispatch runs and Python version assertion says "empty != 3.11".

**How to avoid:** Add `actionlint` to the `lint-workflow` job. It type-checks every `${{ }}` expression against the workflow schema and catches typos before any dispatch.

```bash
# In lint-workflow job:
- name: Install actionlint
  run: |
    bash <(curl https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
- name: Run actionlint on Windows workflow
  run: ./actionlint .github/workflows/build-wheels-cuda-windows.yaml
```

**Warning signs:**
- First-dispatch failures that read like "input x is empty" when it's obviously set.
- Expression results that surprise you (`${{ steps.build.outputs.wheel }}` empty when you expect a filename).

### Pitfall 7: `actions-gh-pages` `permissions` Needed But Not Phase 1's Concern

**What goes wrong:** Phase 4 publish job needs `permissions: contents: write` to push to `gh-pages`. Phase 1 shouldn't set this — it only reads.

**How to avoid:** Phase 1's `permissions:` block is `contents: read` (or omitted, since default is read). Phase 4 will add a job-scoped `permissions:` override for the publish job. **Phase 1 note:** do not set `permissions:` at workflow-level with `contents: write` just because it feels safer — principle of least privilege applies, and wrong permissions can mask bugs (e.g., a `git push` would silently succeed in Phase 1, which would be confusing).

**Warning signs:** None in Phase 1. Included here so the planner doesn't over-scope.

---

## Code Examples

Verified patterns from official sources and cross-referenced with the upstream workflow (`.github/workflows/build-wheels-cuda.yaml`, lines numbered in references).

### Workflow trigger & inputs (WF-02, WF-03, WF-04)

```yaml
# DOC-04: Inline comment block explaining why this workflow exists and the
# msvc_toolset escape hatch. Link upstream #1543 (segfault root cause) and
# upstream #2136 (maintainer status).
on:
  workflow_dispatch:
    inputs:
      python_version:
        description: 'Python version (CPython tag for the wheel)'
        type: choice
        default: '3.11'
        options:
          - '3.10'
          - '3.11'
          - '3.12'
      cuda_version:
        description: 'CUDA toolkit version (must match nvcc major.minor on runner)'
        type: choice
        default: '12.4.1'
        options:
          - '12.4.1'
      msvc_toolset:
        description: 'MSVC toolset MAJOR.MINOR (14.39 = VS 17.9, _MSC_VER 1939; raise only to work around runner image rotation — see upstream #1543)'
        type: string
        default: '14.39'
      verbose:
        description: 'Enable verbose preflight tracing (sets $env:VERBOSE=1)'
        type: boolean
        default: false
      dry_run:
        description: 'Exit green after preflight; skip downstream build (Phase 2+)'
        type: boolean
        default: false
```

### LongPathsEnabled (TC-09)

```yaml
- name: Enable Windows long paths
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'
    # TC-09: avoids MSVC "fatal error C1083: cannot open compiler generated file"
    # when build/{wheel tag}/vendor/llama.cpp/ggml-cuda paths exceed 260 chars.
    New-ItemProperty `
      -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' `
      -Name 'LongPathsEnabled' `
      -Value 1 `
      -PropertyType DWORD `
      -Force | Out-Null
    git config --system core.longpaths true
    Write-Host "LongPathsEnabled: $((Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem').LongPathsEnabled)"

- uses: actions/checkout@v4
  with:
    submodules: recursive   # WF-05
```

### MSVC 14.39 probe + install (TC-02)

```yaml
- name: Probe and install MSVC ${{ inputs.msvc_toolset }} toolset
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    $toolset = '${{ inputs.msvc_toolset }}'
    # Component ID format: Microsoft.VisualStudio.Component.VC.<major>.<minor>.17.9.x86.x64
    # For 14.39 → "Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64"
    # NOTE: the 17.9 suffix is the VS version that originally shipped this toolset;
    # 14.40 uses 17.10, 14.41 uses 17.11, etc. If msvc_toolset bumps, this suffix
    # map needs updating (Phase 1 locks 14.39 as default; MX-02 v2 revisits).
    $componentId = "Microsoft.VisualStudio.Component.VC.$toolset.17.9.x86.x64"

    $vswhere = 'C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe'

    # Probe: is the component offerable on this image?
    $installPath = & $vswhere -latest -property installationPath
    if (-not $installPath) {
      throw "No VS 2022 installation found on windows-2022 runner (vswhere returned empty)"
    }

    $hasComponent = & $vswhere -products * -requires $componentId -property installationPath
    if (-not $hasComponent) {
      Write-Error @"
MSVC toolset $toolset component is NOT available on this windows-2022 image.
Component probed: $componentId

REMEDIATION:
  - Re-dispatch with msvc_toolset=14.40 (works with CUDA 12.5+)
  - See actions/runner-images#9701 for VC component removal history
  - See upstream abetlen/llama-cpp-python#1543 for _MSC_VER/nvcc compatibility
"@
      exit 1
    }

    # Install: modify VS to add the requested toolset component
    $installer = Join-Path $installPath '..\..\..\..\Installer\setup.exe' | Resolve-Path
    Write-Host "Starting vs_installer.exe modify to add $componentId ..."
    $proc = Start-Process -Wait -PassThru -FilePath $installer -ArgumentList @(
      'modify',
      '--installPath', "`"$installPath`"",
      '--add', $componentId,
      '--quiet',
      '--norestart'
    )
    if ($proc.ExitCode -ne 0 -and $proc.ExitCode -ne 3010) {
      # 3010 = success-with-reboot-required, acceptable on CI
      throw "vs_installer.exe modify failed with exit code $($proc.ExitCode)"
    }

    # Re-probe: confirm component is now installed on disk
    $confirmPath = & $vswhere -products * -requires $componentId -property installationPath
    if (-not $confirmPath) {
      throw "Post-install probe failed: $componentId still not present. Installer may have silently failed."
    }
    Write-Host "MSVC $toolset component installed at: $confirmPath"
```

### MSVC activation (TC-03)

```yaml
- name: Activate MSVC ${{ inputs.msvc_toolset }} toolset
  uses: ilammy/msvc-dev-cmd@v1
  with:
    toolset: ${{ inputs.msvc_toolset }}
    vs-version: '[17.9,17.10)'   # DOC-04: tight pin; VS 17.10+ triggers preflight and forces re-validation
    arch: x64
```

### CUDA install (TC-05)

```yaml
- name: Setup mamba
  uses: conda-incubator/setup-miniconda@v3.1.0
  with:
    activate-environment: llamacpp
    python-version: ${{ inputs.python_version }}
    miniforge-version: latest
    add-pip-as-python-dependency: true
    auto-activate-base: false

- name: Install CUDA toolkit via mamba
  shell: bash -el {0}   # -l for conda init; -e for fail-fast
  run: |
    set -euo pipefail
    CUDA_VERSION='${{ inputs.cuda_version }}'
    CUDA_CHANNEL="nvidia/label/cuda-${CUDA_VERSION}"
    # TC-05: single install path. No Jimver full-installer.
    # --channel-priority strict prevents conda-forge from pulling a mismatched cudart.
    mamba install -y \
      --channel-priority strict \
      --override-channels \
      -c "${CUDA_CHANNEL}" \
      "${CUDA_CHANNEL}::cuda-toolkit=${CUDA_VERSION}"
```

### Assertion 1 — MSVC toolset (TC-04)

```yaml
- name: 'Preflight: Assert MSVC _MSC_VER ≤ cap'
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    $toolset = '${{ inputs.msvc_toolset }}'
    $parts = $toolset -split '\.'
    if ($parts.Count -ne 2) { throw "msvc_toolset must be 'MAJOR.MINOR'; got '$toolset'" }
    $major = [int]$parts[0]; $minor = [int]$parts[1]
    if ($major -ne 14) { throw "msvc_toolset major must be 14 for v1 (got '$major')" }
    $cap = 1900 + $minor

    # cl.exe writes banner to stderr; redirect 2>&1.
    $banner = (cl.exe 2>&1 | Out-String)
    $match = [regex]::Match($banner, 'Version (\d+)\.(\d+)')
    if (-not $match.Success) { throw "Failed to parse cl.exe banner: $banner" }
    $clMajor = [int]$match.Groups[1].Value
    $clMinor = [int]$match.Groups[2].Value
    # cl.exe reports '19.39'; _MSC_VER is '19XX' not literally 'X.Y'.
    # _MSC_VER = clMajor * 100 + clMinor when clMajor=19 (e.g. 19.39 → 1939).
    $mscVer = $clMajor * 100 + $clMinor

    if ($mscVer -gt $cap) {
      Write-Error @"
MSVC _MSC_VER=$mscVer exceeds cap=$cap (derived from msvc_toolset=$toolset).

This means ilammy/msvc-dev-cmd did NOT downgrade the active toolset correctly,
OR the vs_installer.exe modify silently failed to install the requested component.

REMEDIATION:
  1. Check the 'Probe and install MSVC' step log for silent failures
  2. Re-run with msvc_toolset=14.40 if 14.39 has been retired from the image
  3. Verify the active toolset: cl.exe /Bv in a debug step
  4. See upstream abetlen/llama-cpp-python#1543 for why _MSC_VER > cap produces segfaulting wheels
"@
      exit 1
    }
    Write-Host "OK: _MSC_VER=$mscVer ≤ cap=$cap"
    Write-Output "msc_ver=$mscVer" >> $env:GITHUB_OUTPUT
```

### Assertion 2 — CUDA toolkit (TC-06)

```yaml
- name: 'Preflight: Assert nvcc matches cuda_version'
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    $requestedFull = '${{ inputs.cuda_version }}'  # e.g. '12.4.1'
    $requestedMajorMinor = ($requestedFull -split '\.')[0..1] -join '.'  # '12.4'

    $nvccOut = (nvcc --version) -join "`n"
    $match = [regex]::Match($nvccOut, 'release (\d+\.\d+)')
    if (-not $match.Success) { throw "Failed to parse 'nvcc --version':`n$nvccOut" }
    $nvccMajorMinor = $match.Groups[1].Value  # '12.4'

    if ($nvccMajorMinor -ne $requestedMajorMinor) {
      throw @"
nvcc reports '$nvccMajorMinor' but dispatch input cuda_version='$requestedFull' (major.minor=$requestedMajorMinor).

REMEDIATION:
  - The mamba install step failed or was skipped
  - A stale CUDA is winning PATH order (see Assertion 3)
  - The cuda_version input is malformed
"@
    }
    Write-Host "OK: nvcc reports $nvccMajorMinor matches requested $requestedMajorMinor"
    Write-Output "nvcc_version=$nvccMajorMinor" >> $env:GITHUB_OUTPUT
```

### Assertion 3 — single nvcc path + no `CUDA_PATH_V*_*` (TC-07)

```yaml
- name: 'Preflight: Assert single CUDA install path'
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    # TC-07: sweep ghost env vars from prior installs
    $ghosts = Get-ChildItem env: | Where-Object { $_.Name -match '^CUDA_PATH_V\d+_\d+$' }
    if ($ghosts.Count -gt 0) {
      foreach ($g in $ghosts) {
        Write-Warning "Removing ghost env var: $($g.Name)=$($g.Value)"
        Remove-Item "env:$($g.Name)"
      }
    }

    # Single nvcc on PATH
    $nvccPaths = (where.exe nvcc 2>$null) -split "`n" | Where-Object { $_ -ne '' }
    if ($nvccPaths.Count -ne 1) {
      throw @"
Expected exactly one nvcc on PATH; found $($nvccPaths.Count):
$($nvccPaths -join "`n")

REMEDIATION:
  - Ensure mamba env is activated (conda activate llamacpp)
  - Remove any Jimver-extracted CUDA from prior job state
"@
    }

    # Ensure the one nvcc lives in CONDA_PREFIX
    if (-not $nvccPaths[0].StartsWith($env:CONDA_PREFIX)) {
      throw "nvcc at $($nvccPaths[0]) is not under CONDA_PREFIX=$env:CONDA_PREFIX"
    }
    Write-Host "OK: single nvcc at $($nvccPaths[0])"
```

### Assertion 4 — submodule populated (WF-05)

```yaml
- name: 'Preflight: Assert llama.cpp submodule populated'
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    if (-not (Test-Path 'vendor/llama.cpp/CMakeLists.txt')) {
      throw @"
vendor/llama.cpp/CMakeLists.txt is missing after 'submodules: recursive' checkout.

REMEDIATION:
  - Re-dispatch the workflow (most likely a transient network failure)
  - If persistent: check .gitmodules and actions/checkout@v4 logs for warnings
  - Verify the submodule isn't path-excluded by .git/info/sparse-checkout
"@
    }
    $llamaSha = (git -C vendor/llama.cpp rev-parse --short HEAD).Trim()
    Write-Host "OK: llama.cpp submodule at $llamaSha"
    Write-Output "llama_sha=$llamaSha" >> $env:GITHUB_OUTPUT
```

### Assertion 5 — Python version match (WF-03)

```yaml
- name: 'Preflight: Assert Python version'
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    $requested = '${{ inputs.python_version }}'  # e.g. '3.11'
    $actualRaw = (python --version 2>&1) -as [string]
    $match = [regex]::Match($actualRaw, 'Python (\d+\.\d+)')
    if (-not $match.Success) { throw "Failed to parse: $actualRaw" }
    $actual = $match.Groups[1].Value

    if ($actual -ne $requested) {
      throw @"
Python version mismatch: requested=$requested, actual=$actual

REMEDIATION:
  - 'where.exe python' may return multiple paths; mamba's Python must win
  - setup-python cache may have served a stale interpreter
  - Check that setup-miniconda activate-environment.python-version matches inputs.python_version
"@
    }
    $pyPath = (where.exe python | Select-Object -First 1).Trim()
    Write-Host "OK: Python $actual at $pyPath"
    Write-Output "python_version=$actual" >> $env:GITHUB_OUTPUT
    Write-Output "python_path=$pyPath" >> $env:GITHUB_OUTPUT
```

### Dynamic VS BuildCustomizations discovery (TC-08)

```yaml
- name: 'Preflight: Discover VS 2022 BuildCustomizations path'
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    # TC-08: dynamic discovery, no hardcoded edition/version
    $vswhere = 'C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe'
    $vsRoot = & $vswhere -latest -property installationPath
    if (-not $vsRoot) { throw "No VS 2022 installation found (vswhere returned empty)" }

    $bcPattern = Join-Path $vsRoot 'MSBuild\Microsoft\VC\*\BuildCustomizations'
    $bcPaths = Get-ChildItem -Path $bcPattern -ErrorAction SilentlyContinue
    if (-not $bcPaths -or $bcPaths.Count -eq 0) {
      throw @"
No VS 2022 BuildCustomizations path found under: $bcPattern

This usually means the runner image has rotated VS editions or the VC++ workload is not installed.

REMEDIATION:
  - Verify windows-2022 runner image still ships VS 2022 Enterprise/Community/BuildTools
  - Check actions/runner-images Windows2022-Readme.md for layout changes
"@
    }
    $bcPath = $bcPaths[0].FullName
    Write-Host "OK: VS BuildCustomizations at $bcPath"
    Write-Output "vs_install_path=$vsRoot" >> $env:GITHUB_OUTPUT
    Write-Output "bc_path=$bcPath" >> $env:GITHUB_OUTPUT
```

### Forensics summary on green

```yaml
- name: 'Preflight: Emit forensics summary'
  if: success()
  shell: pwsh
  run: |
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'

    $osInfo = Get-ComputerInfo -Property OsName,OsBuildNumber
    $cDrive = Get-PSDrive C
    $mambaVer = (mamba --version | Select-Object -First 1).Trim()

    $summary = @"
## Preflight Green — Toolchain Fingerprint

| Property | Value |
|---|---|
| Runner OS | $($osInfo.OsName) build $($osInfo.OsBuildNumber) |
| VS install path | ${{ steps.discover-vs.outputs.vs_install_path }} |
| MSVC toolset | ${{ inputs.msvc_toolset }} (_MSC_VER=${{ steps.assert-msvc.outputs.msc_ver }}) |
| nvcc version | ${{ steps.assert-nvcc.outputs.nvcc_version }} |
| CUDA path | $env:CUDA_PATH |
| Python | ${{ steps.assert-python.outputs.python_version }} at ${{ steps.assert-python.outputs.python_path }} |
| llama.cpp SHA | ${{ steps.assert-submodule.outputs.llama_sha }} |
| mamba version | $mambaVer |
| Disk free (C:) | $([math]::Round($cDrive.Free / 1GB, 2)) GB |
| CPU | $env:PROCESSOR_IDENTIFIER |

**Dispatch inputs:**
- python_version=${{ inputs.python_version }}
- cuda_version=${{ inputs.cuda_version }}
- msvc_toolset=${{ inputs.msvc_toolset }}
- dry_run=${{ inputs.dry_run }}
- verbose=${{ inputs.verbose }}
"@
    $summary | Out-File -FilePath $env:GITHUB_STEP_SUMMARY -Append -Encoding utf8
    Write-Host $summary
```

### Ban-grep self-check (TC-10)

```yaml
lint-workflow:
  runs-on: ubuntu-latest
  timeout-minutes: 5
  steps:
    - uses: actions/checkout@v4

    # TC-10: hard-fail if the banned flag ever lands in our Windows workflow.
    # DOC-04: rationale — '-allow-unsupported-compiler' bypasses nvcc's _MSC_VER
    # check; produces wheels that compile but segfault at first Llama(...) call
    # (upstream abetlen/llama-cpp-python#1543). This is a grep-assert, not a warn.
    - name: 'Lint: ban --allow-unsupported-compiler'
      run: |
        set -euo pipefail
        WORKFLOW='.github/workflows/build-wheels-cuda-windows.yaml'
        if grep -nH 'allow-unsupported-compiler' "$WORKFLOW"; then
          echo "::error file=$WORKFLOW::'-allow-unsupported-compiler' is banned (upstream #1543)"
          exit 1
        fi
        echo "OK: no --allow-unsupported-compiler in $WORKFLOW"

    # Claude's discretion: actionlint as a cheap second lint in the same job.
    # CONTEXT.md deferred a standalone actionlint workflow to v2 AUT-*, but
    # adding it as one extra step here costs ~5s and catches expression typos
    # that ban-grep can't see. Planner decision.
    - name: 'Lint: actionlint'
      run: |
        set -euo pipefail
        bash <(curl -sSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
        ./actionlint -color .github/workflows/build-wheels-cuda-windows.yaml
```

### Inline documentation comments (DOC-04)

```yaml
# Top-of-file preamble:
#
# Windows x64 CUDA wheel builder for llama-cpp-python (fork).
#
# This workflow exists because upstream abetlen/llama-cpp-python disabled
# Windows CUDA CI in July 2025 (commit 98fda8c) after chronic VS/nvcc
# incompatibilities (#1543, #1551, #1838, #1894). The root cause is nvcc's
# strict _MSC_VER check: CUDA 12.4.1 requires _MSC_VER in [1910, 1950), but
# GitHub's windows-2022 image ships VS 17.14 with _MSC_VER=1950. Upstream's
# workaround was -DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler, which
# silences the compile error but produces wheels that segfault at runtime
# on the first Llama(...) call (#1543 reporter confirmed).
#
# This workflow:
#   1. Side-by-side installs MSVC 14.39 (_MSC_VER=1939) on top of the runner's
#      VS 17.14, via vs_installer.exe modify --add VC.14.39.17.9.x86.x64
#   2. Activates the 14.39 toolset via ilammy/msvc-dev-cmd (vcvarsall.bat
#      -vcvars_ver=14.39)
#   3. Installs CUDA 12.4.1 via mamba (single install path — no parallel
#      Jimver full-installer; see upstream Jimver #382)
#   4. Preflights the toolchain with hard-fail assertions (no warn-and-continue)
#   5. Bans -allow-unsupported-compiler via a parallel grep self-check
#
# Current status: Phase 1 (toolchain pinning). Phase 2 adds the build body
# (BLD-01..13). Phase 3 adds smoke-test (ST-*). Phase 4 adds publish (PUB-*).
#
# See .planning/phases/01-scaffold-toolchain-pinning/ for context.
```

---

## State of the Art

| Old Approach (upstream / historical) | Current Approach (this phase) | Impact |
|---|---|---|
| `microsoft/setup-msbuild@v2 vs-version: '[16.11,16.12)'` (upstream line 50) | `ilammy/msvc-dev-cmd@v1 toolset: 14.39 vs-version: '[17.9,17.10)'` | `setup-msbuild` only selects installed VS versions; can't activate a specific MSVC toolset. `ilammy/msvc-dev-cmd` wraps `vcvarsall.bat -vcvars_ver=...` which is the actual working mechanism. |
| `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` (upstream line 159) | Toolset pin + hard-fail preflight + ban-grep | Upstream's flag silences the check; this phase *pins the toolset* so the check passes honestly. Grep-assert prevents regression. |
| Hardcoded `C:\Program Files (x86)\...\2019\Enterprise\...` (upstream line 92) | `vswhere -latest -property installationPath` + glob `$vsRoot\MSBuild\Microsoft\VC\*\BuildCustomizations` | Upstream no-ops silently on `windows-2022` (2019 path doesn't exist). Dynamic detection + strict-mode error flips to loud failure. |
| Two CUDA install paths (Jimver for MSBuildExt + mamba for runtime) | Single path: mamba `cuda-toolkit` | Upstream had `CUDA_PATH` race risk and `cuda-nvcc` / `cuda-cudart` version drift on Windows. TC-05 locks to one path. |
| `shell: pwsh` without strict-mode | `shell: pwsh` + `Set-StrictMode -Version Latest` + `$ErrorActionPreference = 'Stop'` at step head | Default PowerShell silently propagates `$null`; upstream's MSBuildExtensions copy step silently no-ops on the 2019→2022 path mismatch. Strict-mode flips these to loud errors. |
| Monolithic "run all checks" step | One named step per assertion | Upstream pattern makes a single red step mean "one of 5 checks failed, re-read logs to find which." Named steps put the exact failure on the Actions UI. |
| Hardcoded `_MSC_VER` cap of 1939 | `1900 + minor(msvc_toolset)` derivation | Caps the whole toolset policy from a single dispatch input, so bumping the input bumps the cap atomically. |

**Deprecated / outdated items to avoid:**

- **`microsoft/setup-msbuild@v2` for toolset selection** — valid for finding `msbuild.exe`, not for toolset pinning. Don't use it as the sole toolset mechanism.
- **`Jimver/cuda-toolkit@master`** — supply-chain risk. Pin to `@v0.2.24` if ever needed (Phase 2 decision; Phase 1 doesn't use Jimver at all).
- **`actions/setup-python@v4` or older** — Node 16 deprecated. Use `@v5`.
- **`actions/cache@v3`** — backend sunset Feb 2025. (Phase 1 doesn't use caches; note for Phase 2.)
- **`windows-latest` label** — rotating to `windows-2025`. Always pin explicitly.
- **`force_orphan: true` on `peaceiris/actions-gh-pages`** — not Phase 1's concern, but noting it here so the planner keeps it in mind for Phase 4.

---

## Open Questions

1. **Is `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` currently installable on the `windows-2022` runner image via `vs_installer.exe modify`?**
   - **What we know:** Microsoft lists the component as "out of support" (April 2026). `actions/runner-images #9701` started removing redundant VC components in May 2024. The component may still be installable (installer pulls from archive) or may have been purged.
   - **What's unclear:** Empirically, on today's image, does `vs_installer.exe modify --add` succeed?
   - **Recommendation:** The probe-first-install-second pattern (Pattern 2) handles both outcomes correctly. If the component is unavailable, preflight exits immediately at the probe step with a remediation hint pointing at `msvc_toolset: 14.40` + CUDA 12.6.1 (v2 MX-02). **Plan should NOT include "validate component availability" as a separate step** — the probe step already does this inline. First dispatch after Phase 1 ships will answer this empirically.

2. **Does `hashFiles('vendor/llama.cpp/.git/HEAD')` return a stable value when `.git` is a file (git-dir indirection) vs a directory?**
   - **What we know:** `actions/checkout@v4` with `submodules: recursive` typically writes `.git` inside submodules as a *file* pointing into `.git/modules/vendor/llama.cpp/`, not as a directory. `hashFiles` is documented to hash file *contents* regardless of file/directory nature.
   - **What's unclear:** Whether the `.git` *file* content (which is `gitdir: ../../.git/modules/vendor/llama.cpp`) changes when the submodule SHA changes — or only when the indirection path itself changes.
   - **Recommendation:** Phase 1 doesn't use caches (no `hashFiles` calls in Phase 1's YAML), so **this question has zero impact on Phase 1**. It becomes Phase 2's problem for sccache key composition. Phase 1 can emit the submodule SHA into the forensics summary via `git -C vendor/llama.cpp rev-parse --short HEAD`, which is unambiguous. Flagged here so Phase 2 research can follow up.

3. **Should the 14.39 probe-fail path auto-suggest 14.40, or require operator judgment?**
   - **What we know:** 14.40 works with CUDA 12.5+ (raised `_MSC_VER` ceiling to 1950). 14.40 with CUDA 12.4.1 is *reported* working in one NVIDIA-forum source but not empirically validated by us.
   - **What's unclear:** Is bumping only `msvc_toolset` to 14.40 (without bumping `cuda_version` to 12.6.1) a safe 3am-on-call recommendation, or does it introduce a subtly-incompatible combo?
   - **Recommendation:** The remediation hint should suggest **both** `msvc_toolset=14.40` **and** `cuda_version=12.6.1` as a pair (even though `cuda_version` currently has only `12.4.1` in its `choice` options). This is a v2 MX-02 concern — Phase 1's remediation hint points operators at the right doc reference; doesn't try to solve the bump itself.

4. **Should `actionlint` be included in Phase 1's `lint-workflow` job or deferred?**
   - **What we know:** CONTEXT.md deferred a *standalone* actionlint workflow to v2 AUT-*. But Phase 1 already has a `lint-workflow` job with ~5 minutes budget and Ubuntu runner spin-up already paid.
   - **What's unclear:** Whether the planner wants to expand Phase 1's scope by one step (~5s runtime, one extra tool dependency).
   - **Recommendation:** **Include `actionlint` as one extra step in the existing `lint-workflow` job.** Marginal cost is near-zero, it catches a class of bugs ban-grep cannot (expression typos, invalid runner labels), and it's installed via a one-line `curl | bash` (no GitHub Action dependency to pin). This is Claude's discretion per CONTEXT.md. Planner should confirm.

---

## Validation Architecture

### Test Framework

This is a **CI-workflow phase**. There is no traditional unit-test framework — the "test infrastructure" is the `workflow_dispatch` of the workflow itself. Validation of this phase's deliverable happens via:

| Property | Value |
|---|---|
| Primary test framework | **GitHub Actions workflow dispatch** (manual run via Actions UI or `gh workflow run`) |
| Static analyzers | `actionlint` (workflow schema + expressions), `grep` (literal ban-check) |
| Config file | `.github/workflows/build-wheels-cuda-windows.yaml` (the deliverable itself) |
| Quick local validation | `gh workflow run build-wheels-cuda-windows.yaml --ref main -f python_version=3.11 -f cuda_version=12.4.1 -f dry_run=true` |
| Full validation | Same as quick; no "full suite" distinction for Phase 1 |
| Lint-only validation | `actionlint .github/workflows/build-wheels-cuda-windows.yaml` (runnable locally in < 1s) |

The `dry_run` input (CONTEXT.md-locked) lets the workflow exit green immediately after preflight once Phase 2 adds a downstream build job — in Phase 1, `dry_run` has nothing downstream to skip, so it's effectively a no-op passthrough. But the input must be defined now so Phase 2's job-level `if: !inputs.dry_run` works without a workflow-file edit.

### Phase Requirements → Test Map

Every requirement is validated by either (a) a preflight assertion in the workflow itself, or (b) a lint step, or (c) post-dispatch inspection of the workflow log. There is no mock-and-unit-test surface; CI workflows are validated by running them.

| Req ID | Behavior | Test Type | Automated Validation | File Exists? |
|---|---|---|---|---|
| WF-01 | New workflow file exists at correct path | static | `test -f .github/workflows/build-wheels-cuda-windows.yaml` | ❌ Wave 0 creates it |
| WF-02 | Only `workflow_dispatch` triggers | static | `actionlint` + manual YAML inspection for `on:` block | ❌ Wave 0 |
| WF-03 | `python_version` input with default `3.11` | dispatch | Dispatch with no inputs; Actions UI shows dropdown with 3.11 selected | ❌ Wave 0 |
| WF-04 | `cuda_version` input with default `12.4.1` | dispatch | Dispatch with no inputs; Actions UI shows single-option dropdown | ❌ Wave 0 |
| WF-05 | `submodules: recursive` populates `vendor/llama.cpp/` | dispatch (preflight Assertion 4) | `Test-Path vendor/llama.cpp/CMakeLists.txt` in preflight | ❌ Wave 0 |
| TC-01 | `runs-on: windows-2022` literal | static | `grep -nH 'runs-on: windows-2022' .github/workflows/build-wheels-cuda-windows.yaml` | ❌ Wave 0 |
| TC-02 | MSVC 14.39 side-by-side installed | dispatch | Probe-and-install step; post-install `vswhere` re-probe confirms | ❌ Wave 0 |
| TC-03 | `ilammy/msvc-dev-cmd@v1 toolset: 14.39` | static + dispatch | `grep 'ilammy/msvc-dev-cmd'` present; preflight cl.exe banner confirms 14.39 active | ❌ Wave 0 |
| TC-04 | `_MSC_VER ≤ cap` assertion hard-fails on mismatch | dispatch (preflight Assertion 1) | Dispatch with `msvc_toolset: 14.01` (intentionally too-low); assert step turns red | ❌ Wave 0 (negative test is manual, one-shot) |
| TC-05 | Single mamba install path | static + dispatch | `grep 'mamba install'` in exactly one step; no `Jimver/cuda-toolkit` uses block | ❌ Wave 0 |
| TC-06 | `nvcc --version` matches input | dispatch (preflight Assertion 2) | Preflight step compares `nvcc --version` regex to `inputs.cuda_version` | ❌ Wave 0 |
| TC-07 | No `CUDA_PATH_V*_*` survives + single `where nvcc` | dispatch (preflight Assertion 3) | Preflight sweep + `where.exe nvcc` count-one assertion | ❌ Wave 0 |
| TC-08 | Dynamic VS BuildCustomizations discovery | dispatch | Preflight discovery step; strict-mode error if empty match | ❌ Wave 0 |
| TC-09 | `LongPathsEnabled=1` before checkout | dispatch | Explicit step before `actions/checkout`; `Get-ItemProperty` confirms set | ❌ Wave 0 |
| TC-10 | `-allow-unsupported-compiler` banned (grep) | static (`lint-workflow`) | `lint-workflow` job runs grep on every dispatch; also runnable locally | ❌ Wave 0 |
| DOC-04 | YAML inline comments explain rationale | manual (code review) | Human review of YAML comment coverage; `grep '#.*1543'` for #1543 link | ❌ Wave 0 (review-only) |

**"Wave 0 gaps" column interpretation:** For a CI workflow phase, the entire deliverable *is* Wave 0. There's no pre-existing test infrastructure to slot into; we're creating the workflow from scratch. Every row is ❌ because the workflow file doesn't exist yet at phase start.

### Sampling Rate

- **Per task commit (during Wave 0 implementation):** `actionlint .github/workflows/build-wheels-cuda-windows.yaml` (local, < 1s) + `grep -n 'allow-unsupported-compiler' .github/workflows/build-wheels-cuda-windows.yaml` (local, < 1s). These replicate the `lint-workflow` job locally without a dispatch.
- **Per wave merge:** One `workflow_dispatch` with defaults (`python_version=3.11`, `cuda_version=12.4.1`, `msvc_toolset=14.39`, `dry_run=true`). Validates the full preflight path end-to-end on a real runner.
- **Phase gate (`/gsd:verify-work`):** Above *plus* one negative-case dispatch: `msvc_toolset=14.01` (below floor) — confirms Assertion 1 hard-fails with remediation hint, not silent pass. Then a cleanup dispatch returning to defaults confirms the negative-test didn't break state.

**Budget:** 3 dispatches max per phase gate. Each dispatch is ~2-5 min on `windows-2022` runner minutes (preflight is cheap; no build). Tolerable.

### Wave 0 Gaps

The workflow file does not exist at phase start. Every deliverable is a Wave 0 creation:

- [ ] `.github/workflows/build-wheels-cuda-windows.yaml` — the workflow itself (WF-01)
- [ ] Top-of-file YAML preamble comment block explaining fork rationale + upstream issue links (DOC-04)
- [ ] `on: workflow_dispatch:` block with 5 inputs (WF-02, WF-03, WF-04, plus `msvc_toolset`, `verbose`, `dry_run`)
- [ ] `permissions: contents: read` (principle of least privilege for Phase 1)
- [ ] `preflight` job skeleton (TC-01 `runs-on: windows-2022`, `defaults: run: shell: pwsh`, `timeout-minutes: 20`)
- [ ] `LongPathsEnabled` registry step (TC-09)
- [ ] `actions/checkout@v4` with `submodules: recursive` (WF-05)
- [ ] `actions/setup-python@v5` step with `python-version: ${{ inputs.python_version }}`
- [ ] MSVC 14.39 probe-and-install step (TC-02, uses `vswhere` + `vs_installer.exe modify`)
- [ ] `ilammy/msvc-dev-cmd@v1` activation step (TC-03)
- [ ] `conda-incubator/setup-miniconda@v3.1.0` + mamba install step (TC-05)
- [ ] 5 preflight assertion steps, one per assertion (TC-04, TC-06, TC-07, WF-05, Python version check)
- [ ] Dynamic VS BuildCustomizations discovery step (TC-08)
- [ ] Forensics `$GITHUB_STEP_SUMMARY` emit step on green
- [ ] `lint-workflow` job skeleton (`runs-on: ubuntu-latest`, `timeout-minutes: 5`)
- [ ] `ban-grep` step for `-allow-unsupported-compiler` (TC-10)
- [ ] *(Claude's discretion)* `actionlint` step in `lint-workflow` job

No pre-existing test infrastructure to slot into. No framework-install steps needed (runner pre-ships `vswhere`, `vs_installer.exe`, `cl.exe`; `ilammy/msvc-dev-cmd` is a marketplace action; `actionlint` is a one-line `curl | bash` install in the lint job).

---

## Sources

### Primary (HIGH confidence)

- **NVIDIA CUDA Installation Guide for Microsoft Windows 12.4.1** — https://docs.nvidia.com/cuda/archive/12.4.1/cuda-installation-guide-microsoft-windows/index.html — `_MSC_VER` whitelist, host compiler requirements
- **Microsoft devblogs — "MSVC Toolset Minor Version Number 14.40 in VS 2022 v17.10"** — https://devblogs.microsoft.com/cppblog/msvc-toolset-minor-version-number-14-40-in-vs-2022-v17-10/ — toolset↔`_MSC_VER` formula (14.39 → 1939, 14.40 → 1940), component ID naming convention
- **actions/runner-images Windows2022-Readme.md** — https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md — current VS version (17.14.37111.16, MSVC 14.50), preinstalled tools
- **actions/runner-images #9701** — https://github.com/actions/runner-images/issues/9701 — VC component removal announcement (May 2024); basis for Pitfall 1
- **actions/runner-images #12677** — https://github.com/actions/runner-images/issues/12677 — `windows-latest` repointing to `windows-2025` (TC-01 rationale)
- **ilammy/msvc-dev-cmd README** — https://github.com/ilammy/msvc-dev-cmd — `toolset:` input wraps `vcvarsall.bat -vcvars_ver=...`
- **microsoft/vswhere Wiki** — https://github.com/microsoft/vswhere/wiki — `-products *`, `-requires`, `-latest`, `-property installationPath` flags
- **Microsoft visualstudio-docs — workload-component-id-vs-build-tools.md** — https://github.com/MicrosoftDocs/visualstudio-docs/blob/main/docs/install/includes/vs-2022/workload-component-id-vs-build-tools.md — VS Build Tools component IDs (source for `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` naming)
- **GitHub Actions Changelog — Early February 2026** — https://github.blog/changelog/2026-02-05-github-actions-early-february-2026-updates/ — `windows-2025-vs2026` availability, windows-2025 integration May 2026
- **rhysd/actionlint** — https://github.com/rhysd/actionlint — static analyzer, v1.7.11 (Feb 2026), expression type-checking

### Secondary (MEDIUM confidence)

- **abetlen/llama-cpp-python issue #1543** — https://github.com/abetlen/llama-cpp-python/issues/1543 — root reference for `-allow-unsupported-compiler` segfault; Pitfall 1 and the entire phase's rationale
- **abetlen/llama-cpp-python issue #2136** — https://github.com/abetlen/llama-cpp-python/issues/2136 — upstream maintainer status (Quansight funding offer unanswered); justifies fork
- **Microsoft/STL issue #4941** — https://github.com/microsoft/STL/issues/4941 — VS 17.9 (MSVC 14.39) as recommended for CUDA 12.4
- **nerfstudio-project/nerfstudio issue #3171** — https://github.com/nerfstudio-project/nerfstudio/issues/3171 — documents nvcc error message and side-by-side toolset workaround pattern
- **cupy/cupy issue #8578** — https://github.com/cupy/cupy/issues/8578 — sibling project hitting same VS/nvcc rotation
- **Jimver/cuda-toolkit issue #382** — https://github.com/Jimver/cuda-toolkit/issues/382 — full installer hang on `windows-2022` for CUDA 12.5+ (TC-05 rationale)
- **actions/runner-images #13638** — https://github.com/actions/runner-images/issues/13638 — `windows-2025-vs2026` beta details
- **KirillOsenkov/MSBuildTools — msvc_toolset_cheatsheet.md** — https://github.com/KirillOsenkov/MSBuildTools/blob/main/msvc_toolset_cheatsheet.md — toolset↔VS version mapping cross-check
- **thepwrtank18/install-vs-components** — https://github.com/thepwrtank18/install-vs-components — reference pattern for `vs_installer.exe modify` in CI

### Tertiary (LOW confidence — flagged for empirical validation)

- **Current installability of `Microsoft.VisualStudio.Component.VC.14.39.17.9.x86.x64` on today's `windows-2022` image** — Microsoft catalogs the component as "out of support." Whether `vs_installer.exe modify` can still pull it from archive is empirically time-sensitive. Mitigated by probe-first pattern. **First Phase 1 dispatch will answer this.**
- **14.40 + CUDA 12.4.1 combination** — reported working by one NVIDIA-forum source; not empirically validated here. Remediation hint suggests bumping to 14.40 *and* 12.6.1 as a pair.
- **`vendor/llama.cpp/.git/HEAD` file-vs-directory edge case for `hashFiles`** — Phase 1 doesn't use `hashFiles` so no immediate impact. Deferred to Phase 2 research.

### Local references

- **Upstream workflow** — `C:\claude_checkouts\llama-cpp-python\.github\workflows\build-wheels-cuda.yaml` — pattern reference (lines 46-181), especially `Get Visual Studio Integration` (line 78-86) for the upstream MSBuildExtensions extraction pattern (Phase 2 may reuse)
- **Project research SUMMARY** — `.planning/research/SUMMARY.md` — cross-verifies all stack choices
- **Project research PITFALLS** — `.planning/research/PITFALLS.md` — Pitfalls 1-5, 14, 15 all land in Phase 1 territory
- **Project research STACK** — `.planning/research/STACK.md` — version pins and alternatives rejected
- **Keepassxc reference build** — `C:\claude_checkouts\keepassxc\.github\workflows\build.yml` — *not relevant to Phase 1* (Phase 1 has no caches); flagged for Phase 2 research

---

## Metadata

**Confidence breakdown:**

- **Standard Stack:** HIGH — every version pin cross-referenced to official NVIDIA/Microsoft docs and upstream precedent. Single LOW item: exact behavior of `vs_installer.exe modify` against an "out of support" component on today's image (probe-first pattern makes outcome deterministic either way).
- **Architecture Patterns:** HIGH — two-job layout, probe-first-install-second, strict-mode assertions, step-summary forensics are all standard patterns for robust Windows CI.
- **Pitfalls:** HIGH — 7 pitfalls catalogued, 5 from PITFALLS.md that map to Phase 1, plus 2 new (YAML expression typos, permissions scope confusion). All backed by issues or documented failure modes.
- **Code Examples:** HIGH — every example was either lifted from verified upstream code or composed from documented PowerShell / GitHub Actions primitives. One-shot validation recommended on first dispatch.
- **Validation Architecture:** HIGH — inherently self-validating (the workflow's preflight assertions are the tests). The "negative test" pattern (intentionally-failing dispatch to confirm hard-fail behavior) is a standard CI self-check technique.

**Research date:** 2026-04-15
**Valid until:** 2026-05-15 for the 14.39 component availability (fast-moving, runner-image-rotation dependent); 2026-07-15 for the rest (stable toolchain patterns). Re-validate component availability on first phase gate fail.
