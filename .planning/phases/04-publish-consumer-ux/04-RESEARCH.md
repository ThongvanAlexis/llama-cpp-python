# Phase 4: Publish & Consumer UX - Research

**Researched:** 2026-04-19
**Domain:** GitHub Releases automation, softprops/action-gh-release@v2, GITHUB_TOKEN permission scoping, multi-line release-body assembly in bash, consumer-facing README for wheel-from-Releases install flow
**Confidence:** HIGH

## Summary

Phase 4 adds a `publish` job to the existing `build-wheels-cuda-windows.yaml` workflow on `ubuntu-latest`, gated by `needs: [build, smoke-test]`. The job downloads the `cuda-wheel` artifact (Phase 2 contract), computes metadata, assembles a release-notes body with the llama.cpp submodule SHA forensics table, and uploads the wheel as a GitHub Release asset via `softprops/action-gh-release@v2`. The tag scheme is `v<base_version>-cu126-win` (namespaced for future Linux/macOS coexistence); the action creates the tag on-the-fly via GitHub's REST `createRelease` with `target_commitish: ${{ github.sha }}`. A grep-assert step in the existing `lint-workflow` job enforces `needs: [build, smoke-test]` on the publish job (character-class regex trick from Phase 1 to prevent self-match). The repo README is overwritten with a fork-focused document; upstream README preserved as `README.upstream.md`.

**SCOPE REDUCTION (locked 2026-04-19):** CONTEXT.md dropped PEP 503 / gh-pages / Fastly from Phase 4. Consumers install by downloading the `.whl` from the Releases page and running `pip install path/to/wheel.whl`. Research below is targeted at the reduced scope only — no gh-pages, no `peaceiris/actions-gh-pages`, no `--extra-index-url`, no HTTP probe.

The technical domains are all well-understood and have HIGH-confidence primary sources: softprops v2 is locked and documented (action.yml + README), GitHub API behavior for `createRelease` with `target_commitish` is documented, job-level `permissions:` scoping is documented, and NVIDIA's official CUDA 12.x compatibility matrix gives the authoritative Windows driver floor.

**Primary recommendation:** Implement as **a single plan** (04-01). The publish job, lint-workflow grep-assert, README overwrite, and REQUIREMENTS.md amendment are all touching distinct files with no inter-task dependency beyond "all land in one commit or the `needs: [build, smoke-test]` gate is half-wired." CONTEXT.md explicitly left the 1-vs-2 split to planner's discretion — the scope reduction collapses what was two plans' worth of work into one cohesive change.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**SCOPE REDUCTION — PEP 503 / gh-pages dropped on 2026-04-19.** Original phase scope included a `gh-pages` PEP 503 index with SHA-256 fragments, `data-requires-python` attrs, `peaceiris/actions-gh-pages@v4` + `keep_files: true`, `concurrency: gh-pages-publish`, and a post-publish Fastly probe (PUB-03..10, DOC-03). All dropped. Consumers install by manually downloading the wheel from the Releases page and running `pip install path/to/wheel.whl`. REQUIREMENTS.md needs an amendment to move PUB-03..PUB-10 and DOC-03 to "Out of Scope"; DOC-01's canonical command is reinterpreted as manual download + pip install.

Phase 4 is the last phase of v1 — there are no downstream phases reading this CONTEXT.md inside the current milestone.

#### Release tag & asset scheme
- **Tag format**: `v<base_version>-cu126-win` (e.g. `v0.3.20-cu126-win`). Namespaces Windows so future Linux/Mac wheels can coexist under different tag suffixes. One Release per `(base_version, cu-tag, os)` tuple; assets accumulate across Python versions on the same tag.
- **Asset upload**: `softprops/action-gh-release@v2` (PUB-02). Release is the sole storage location — no gh-pages duplication.
- **Re-dispatch behavior**: clobber. Same wheel filename → overwrite existing asset on the same tag. Latest green build wins for its exact filename slot. Fits the fail-fix-retry CI loop without producing history clutter.
- **Wheel filename**: keep Phase 2's `+cu126.ll<sha12>` local-version segment (`llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl`). No rework of Phase 2's `SETUPTOOLS_SCM_PRETEND_VERSION` logic. SHA-in-filename proves the embedded llama.cpp commit at a glance even when the wheel is archived without the Release context.
- **First-publish bootstrap**: softprops auto-creates the Release on first push of a tag. No manual pre-creation needed.

#### Publish job layout
- **Runner**: `ubuntu-latest` (PUB-01). Cheap minutes; no CUDA or MSVC needed for a Release asset upload.
- **Dependencies**: `needs: [build, smoke-test]` — hard gate. Publish is structurally incapable of running if smoke-test is red.
- **Permissions**: `contents: write` on the publish job only (enables Release asset upload with the auto-provided `GITHUB_TOKEN`; no PAT needed for same-repo pushes).
- **Concurrency**: NONE. `workflow_dispatch` requires Write access, so on a public-but-solo fork only maintainers can trigger parallel runs. User will self-coordinate; accept the near-zero risk of parallel same-input dispatches.
- **Steps**: download `cuda-wheel` artifact (Phase 2 contract) → read wheel filename + compute metadata → construct Release body → softprops upload with `fail_on_unmatched_files: false`. Each step has explicit `timeout-minutes`.

#### Gate assertion (deferred from Phase 3)
- **Location**: extend the existing `lint-workflow` ubuntu job with a new step alongside the ban-grep and actionlint. No new job; no new runner minutes.
- **Check**: grep the workflow YAML for the literal `needs: [build, smoke-test]` line on the publish job; fail with a remediation hint if missing.
- **Pattern**: same character-class-escape trick used by Phase 1's ban-grep (`needs[:].*build.*smoke[-]test` or similar) to avoid self-match on the lint step's own source line.

#### Release body (DOC-05 + forensics)
- **Format**: narrative intro + full forensics table.
- **Intro line**: `Windows CUDA 12.6 wheel, built YYYY-MM-DD against llama.cpp @ <sha12>.`
- **Table columns**: llama.cpp SHA (12-char, required by DOC-05), base version, CUDA version, MSVC toolset selected, Python version, wheel size (bytes / MiB), build date (UTC), dispatch run URL, smoke-test exit code.
- **Generation**: assembled inline in the publish job using outputs from prior jobs (`assert-submodule.outputs.llama_sha`, `probe-msvc.outputs.selected_toolset`, wheel metadata, `github.run_id`, etc.). No external template file.

#### README strategy
- **Treatment**: overwrite. Rename current upstream README to `README.upstream.md` (preserves 824 lines of OpenAI/LangChain/docker/backend content for reference and rebase hygiene), write a fresh fork-focused `README.md`.
- **New README structure** (all one file, top-to-bottom):
  1. **Intro** — one paragraph: "This fork adds a CI workflow to build a Windows x64 CUDA wheel of `llama-cpp-python` with pre-packaged CUDA DLLs (no CUDA install needed on the consumer side)." Explain upstream paused Windows CUDA wheels in July 2025 (abetlen/llama-cpp-python#1543, #2136) and this fork reactivates that path for our own use.
  2. **Install (Windows CUDA)** — numbered steps with prereqs:
     1. Check NVIDIA driver ≥ 560.x via `nvidia-smi` (DOC-02; CUDA 12.6 driver floor)
     2. Go to Releases: `https://github.com/ThongvanAlexis/llama-cpp-python/releases`, pick the latest `v<ver>-cu126-win` tag
     3. Download the `.whl` matching your Python version (cp310 / cp311 / cp312)
     4. `pip install path/to/wheel.whl`
     5. Verify: `python -c "from llama_cpp import Llama; print('ok')"`
  3. **How the wheel is built** — short explanation of the CI pipeline (preflight → build → smoke-test gate → publish), the MSVC/nvcc pinning story, why `-allow-unsupported-compiler` is banned (segfault trap per upstream #1543). Link to `.github/workflows/build-wheels-cuda-windows.yaml` for the authoritative source.
  4. **Attribution** — "Upstream project: `github.com/abetlen/llama-cpp-python`. Original README preserved at `README.upstream.md`."
  5. **License** — carry upstream's MIT license line unchanged.
- **Fork URL**: hardcoded `https://github.com/ThongvanAlexis/llama-cpp-python/...` — README is useless to consumers without the literal URL.
- **DOC-03 (Fastly delay)**: removed; no longer applicable without gh-pages.
- **DOC-01 reinterpreted**: the required "Install (Windows CUDA)" section exists, but its content is the numbered manual-download steps above rather than an `--extra-index-url` one-liner.

#### REQUIREMENTS.md amendments (Plan 4-02 will apply these)
- Move **PUB-03, PUB-04, PUB-05, PUB-06, PUB-07, PUB-08, PUB-09, PUB-10** from the active v1 list to "Out of Scope" with a one-line reason: *"Consumer install via manual download from Releases page; no PEP 503 index on gh-pages in v1. Could be revisited in v2."*
- Move **DOC-03** to "Out of Scope" (no Fastly delay without gh-pages).
- Rewrite **DOC-01** to: *"README `Install (Windows CUDA)` section explains the manual download + `pip install path/to/wheel.whl` flow, points at `https://github.com/ThongvanAlexis/llama-cpp-python/releases`, and lists prereqs including the NVIDIA driver floor."*
- Update **PROJECT.md Core Value** line from `pip install llama-cpp-python --extra-index-url https://...github.io/...` to a manual-install wording. Add a Key Decision row capturing the 2026-04-19 scope reduction rationale ("gh-pages complexity not worth the maintenance cost for a solo fork; Release-only publishing delivers core runtime-works guarantee with less surface area").

### Claude's Discretion
- Exact `softprops/action-gh-release@v2` invocation (asset glob, `fail_on_unmatched_files`, `prerelease` flag — user didn't specify; default to `prerelease: false` since this is the stable consumer-facing asset).
- Exact grep-assert regex and remediation-hint wording (match Phase 1 ban-grep style).
- Release body table assembly: inline bash heredoc in the publish job vs inline `python -c` vs a single `printf`. Whichever reads cleanest under the PROJECT.md constraint ("no shell scripts outside the workflow").
- Per-step `timeout-minutes` values.
- Wheel-size unit formatting in the Release body table (MiB with 1 decimal).
- Exact wording of the README intro paragraph and the "How the wheel is built" section — just hit the key points above.
- Whether to include a `nvidia-smi` example output in the README prereqs step (helpful polish, not load-bearing).
- Whether a CHANGELOG.md stub gets added alongside the README overwrite (surfaced during discussion but not decided; Claude's call — likely skip in v1, revisit in v2).
- Plan-split decision: ROADMAP sketches 2 plans (04-01 publish job + 04-02 README/release notes). Given the scope reduction, a single combined plan may now be appropriate. Planner picks 1 vs 2 based on task dependency graph.

### Deferred Ideas (OUT OF SCOPE)
- **PEP 503 index on gh-pages** — original Phase 4 scope. Dropped 2026-04-19. Rationale: maintenance cost of gh-pages branch + index regeneration + Fastly probe + `--extra-index-url` UX didn't justify the complexity for a solo-maintainer fork. If the fork grows a wider user base or CI matrix, revisit as a standalone phase in v2. When revisited: the requirement set is PUB-03..PUB-10 + DOC-03; the index generator must regenerate from dir listing (not GH API) with SHA-256 fragments and `data-requires-python`; bootstrap first-time gh-pages branch; post-publish Fastly probe with 15×60s timeout.
- **Auto-trigger on tag push / Release publish** — `AUT-01` / `AUT-02`; v2 scope.
- **Sigstore / PEP 740 attestations on uploaded wheels** — `QA-01`; v2 scope.
- **CHANGELOG.md stub** — surfaced during discussion, not decided. Low cost to add a skeleton ("Windows CUDA wheel builds start with v0.3.20-cu126-win"), but not required by any v1 requirement. Planner's call.
- **`nvidia-smi` example output screenshot/snippet in README** — polish, not a requirement. Planner may include a formatted code block showing expected `nvidia-smi` output to make the prereqs step self-verifiable.
- **Multi-Python matrix / multi-CUDA matrix wheels on the same Release tag** — works out of the box with the chosen `v<ver>-cu126-win` scheme (assets accumulate). Not a Phase 4 task to exercise; falls out as a v2 matrix expansion (`MX-01`, `MX-02`).
- **Repository-wide ban-grep as always-on PR check** — still deferred from Phase 1.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PUB-01 | Publish job runs on `ubuntu-latest` | Standard GH Actions `runs-on: ubuntu-latest` — no CUDA needed for Release asset upload; 5× cheaper minutes than windows-2022 |
| PUB-02 | Wheel uploaded as asset on a GitHub Release via `softprops/action-gh-release@v2` | v2 is locked in CONTEXT.md; v2.6.2 is the latest patch (last Node 20-compatible line). v3.0.0 exists (Node 24) but not selected. Action supports `files:` (newline-delimited globs), `tag_name:`, `name:`, `body:`/`body_path:`, `target_commitish:`, `overwrite_files: true` (default), `fail_on_unmatched_files`, `prerelease`, `make_latest`. |
| PUB-03..PUB-10 | gh-pages PEP 503 index requirements | **DEFERRED TO v2** per CONTEXT.md. Not addressed by Phase 4. Plan 04-02 amends REQUIREMENTS.md to move these to Out of Scope. |
| DOC-01 | README "Install (Windows CUDA)" section with canonical install command | **REINTERPRETED** per CONTEXT.md: manual download from Releases + `pip install path/to/wheel.whl` (not `--extra-index-url`). README prereqs + numbered steps + fork URL. |
| DOC-02 | Minimum NVIDIA driver version | **CORRECTED** by research: Windows minor-version-compat floor for CUDA 12.x = **527.41** (R525 branch); CUDA 12.6.3 toolkit-install floor = **561.17** (R560 branch). For a consumer running a 12.6-linked wheel, **527.41** is the absolute minimum. CONTEXT.md's "≥ 560.x" figure is the toolkit-install floor; using **≥ 561.17** in the README is safe and matches what NVIDIA's release notes cite, or alternatively "**≥ 527.41 (CUDA 12.x minor-version compat)** with **≥ 561.17 recommended** (matches build)" for completeness. |
| DOC-03 | Fastly cache delay note | **DEFERRED TO v2** per CONTEXT.md (no gh-pages, no Fastly). Plan 04-02 moves this to Out of Scope. |
| DOC-05 | Release notes record llama.cpp submodule SHA | Release body assembled inline in the publish job. Uses `assert-submodule.outputs.llama_sha` (7-char) + re-derived 12-char SHA for the wheel filename match. Narrative intro line + forensics table. |
</phase_requirements>

## Standard Stack

### Core
| Library/Tool | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `softprops/action-gh-release` | `@v2` (pin floating on v2.6.2 as of 2026-04-12) | Upload wheel as Release asset, create Release + tag on-the-fly | Locked by CONTEXT.md / PUB-02. Upstream already uses it (`build-and-release.yaml:202`). v2 stays on Node 20 runtime (safe for current ubuntu-latest image). |
| `actions/download-artifact` | `@v4` | Download `cuda-wheel` artifact produced by the build job | Phase 2 contract (`actions/upload-artifact@v4` in build job). v4 is the mandatory pair — v3 won't download v4 artifacts. |
| `actions/checkout` | `@v4` | Checkout only needed if publish job reads repo files (unlikely — wheel comes from artifact) | Existing workflow pin. Can be SKIPPED entirely on the publish job since the wheel + all metadata come from artifact + prior-job outputs. |

### Supporting
| Tool | Purpose | When to Use |
|------|---------|-------------|
| `sha256sum` (coreutils) | Compute SHA-256 of the wheel for the release body forensics | Inline in publish job's bash step. Wheel hash is a diagnostic aid, not used for index fragments (those were the dropped PUB-06 scope). |
| `du -b` / `stat -c%s` | Get wheel byte size for the release body table | Inline bash; MiB formatting via `awk` or bash arithmetic. |
| Bash heredoc (`cat <<EOF >> $GITHUB_OUTPUT`) | Multi-line release body via GITHUB_OUTPUT (if passing between steps) — or direct `body:` input | Use random delimiter (not plain `EOF`) to avoid accidental collision with body content containing the literal `EOF`. |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `softprops/action-gh-release@v2` | `softprops/action-gh-release@v3` (latest, Node 24) | v3 released 2026-04-12, works on current runner images, but **CONTEXT.md locks v2**. v2.6.2 will be maintained in parallel. No reason to deviate. |
| `softprops/action-gh-release@v2` | `gh release create` via gh CLI | gh CLI is pre-installed on ubuntu-latest runners; would avoid an external action. Tradeoff: more shell code to maintain; less forensic clarity in Actions UI; upstream's own release path uses softprops so parity wins. Stay with softprops. |
| Job-level `permissions: contents: write` | Workflow-level | Job-level is strictly better (principle of least privilege). Other jobs (build, smoke-test, lint-workflow) should NOT have write access. |
| `target_commitish: ${{ github.sha }}` | Rely on default (repo default branch) | Default is safer when dispatch is run from `main`, but `github.sha` pins the release tag to the exact commit the wheel was built from — critical for reproducibility when dispatches happen from non-default branches. Use `github.sha`. |
| `body_path:` (file) | `body:` (inline string) | Either works. `body:` with a multi-line YAML block scalar is cleaner for the small table this produces; `body_path:` requires emitting a temp file in a prior step. Pick `body:` with `${{ steps.release-body.outputs.content }}` built in a heredoc step. |

**Installation:**
No pip installs needed on the publish job — it is pure YAML + bash. The publish step's runner (ubuntu-latest) ships `gh`, `curl`, `sha256sum`, `jq`, `awk`, `sed` pre-installed.

## Architecture Patterns

### Publish Job Structure

```yaml
publish:
  needs: [build, smoke-test]          # Hard gate — publish is structurally incapable of running without a green smoke-test.
  runs-on: ubuntu-latest
  timeout-minutes: 10                 # Sanity bound — the job should complete in < 2 min.
  permissions:
    contents: write                    # Minimum required by softprops/action-gh-release; all other scopes implicitly `none`.
  defaults:
    run:
      shell: bash
  steps:
    # 1. Download wheel artifact (no checkout needed — wheel + metadata come from prior jobs' outputs)
    - name: 'Publish: Download wheel artifact'
      uses: actions/download-artifact@v4
      with:
        name: cuda-wheel
        path: ./dist
      timeout-minutes: 5

    # 2. Compute release metadata (wheel filename, size, sha256, tag name, date)
    - name: 'Publish: Compute release metadata'
      id: meta
      run: |
        set -euo pipefail
        WHEEL=$(ls dist/*.whl | head -1)
        if [[ -z "$WHEEL" ]]; then
          echo "::error::No wheel found in dist/ after artifact download"
          exit 1
        fi
        WHEEL_NAME=$(basename "$WHEEL")
        WHEEL_SIZE=$(stat -c%s "$WHEEL")
        WHEEL_SIZE_MIB=$(awk -v s="$WHEEL_SIZE" 'BEGIN { printf "%.1f", s/1024/1024 }')
        WHEEL_SHA256=$(sha256sum "$WHEEL" | cut -d' ' -f1)
        BUILD_DATE=$(date -u +%Y-%m-%d)
        # Extract base version (e.g. 0.3.20) from filename llama_cpp_python-0.3.20+cu126.ll<sha>-cp311-...
        BASE_VER=$(echo "$WHEEL_NAME" | sed -E 's/^llama_cpp_python-([0-9.]+)\+.*/\1/')
        TAG="v${BASE_VER}-cu126-win"
        {
          echo "wheel_name=$WHEEL_NAME"
          echo "wheel_path=$WHEEL"
          echo "wheel_size_bytes=$WHEEL_SIZE"
          echo "wheel_size_mib=$WHEEL_SIZE_MIB"
          echo "wheel_sha256=$WHEEL_SHA256"
          echo "build_date=$BUILD_DATE"
          echo "tag=$TAG"
          echo "base_version=$BASE_VER"
        } >> "$GITHUB_OUTPUT"
      timeout-minutes: 2

    # 3. Assemble release body (heredoc; multi-line → GITHUB_OUTPUT with random delimiter)
    - name: 'Publish: Assemble release body'
      id: body
      run: |
        set -euo pipefail
        DELIM="RELEASE_BODY_$(date +%s)_$RANDOM"
        {
          echo "content<<$DELIM"
          cat <<'BODY'
        Windows CUDA 12.6 wheel, built ${{ steps.meta.outputs.build_date }} against llama.cpp @ ${{ needs.build.outputs.llama_sha }}.

        | Property | Value |
        |---|---|
        | llama.cpp SHA | `${{ needs.build.outputs.llama_sha }}` |
        | Base version | `${{ steps.meta.outputs.base_version }}` |
        | CUDA version | `${{ inputs.cuda_version }}` |
        | MSVC toolset | `${{ needs.build.outputs.selected_toolset }}` |
        | Python | `${{ inputs.python_version }}` |
        | Wheel size | `${{ steps.meta.outputs.wheel_size_bytes }}` bytes (${{ steps.meta.outputs.wheel_size_mib }} MiB) |
        | Wheel SHA-256 | `${{ steps.meta.outputs.wheel_sha256 }}` |
        | Build date (UTC) | `${{ steps.meta.outputs.build_date }}` |
        | Dispatch run | https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }} |
        | Smoke-test | green (publish gated on `needs: [build, smoke-test]`) |
        BODY
          echo "$DELIM"
        } >> "$GITHUB_OUTPUT"
      timeout-minutes: 2

    # 4. Upload wheel as Release asset (creates tag + release on first push)
    - name: 'Publish: Upload wheel as GitHub Release asset'
      uses: softprops/action-gh-release@v2
      with:
        tag_name: ${{ steps.meta.outputs.tag }}
        name: ${{ steps.meta.outputs.tag }}
        body: ${{ steps.body.outputs.content }}
        files: dist/*.whl
        target_commitish: ${{ github.sha }}     # Pin tag to exact build commit
        fail_on_unmatched_files: true           # Catch an empty dist/ early
        prerelease: false
        make_latest: 'true'                      # String, not bool — action.yml defines it as string
        # overwrite_files defaults to true — re-dispatch clobbers same-name assets per CONTEXT.md
      timeout-minutes: 5

    # 5. Green forensics summary (matches Phase 1/2/3 pattern)
    - name: 'Publish: Green forensics summary'
      if: success()
      run: |
        cat >> "$GITHUB_STEP_SUMMARY" <<EOF
        ## Publish Green — Wheel uploaded to GitHub Release

        | Property | Value |
        |---|---|
        | Tag | \`${{ steps.meta.outputs.tag }}\` |
        | Wheel | \`${{ steps.meta.outputs.wheel_name }}\` |
        | Size | ${{ steps.meta.outputs.wheel_size_mib }} MiB |
        | SHA-256 | \`${{ steps.meta.outputs.wheel_sha256 }}\` |
        | llama.cpp | \`${{ needs.build.outputs.llama_sha }}\` |
        | Release URL | See Releases page: https://github.com/${{ github.repository }}/releases/tag/${{ steps.meta.outputs.tag }} |
        EOF
      timeout-minutes: 2
```

### Pattern 1: Job-Level Permissions (Least Privilege)
**What:** Set `permissions: contents: write` at the publish job level, NOT workflow level.
**When to use:** Always, for any job that needs a non-default scope. All other permissions implicitly become `none`.
**Example:**
```yaml
# Source: https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#permissions
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write    # Sufficient for softprops/action-gh-release@v2 Release + asset upload
    # build, smoke-test, lint-workflow get default read-only GITHUB_TOKEN
```
**Why:** GitHub's documentation states "If you specify the access for any of these permissions, all of those that are not specified are set to `none`." Workflow-level permissions don't leak to other jobs. This is the principle-of-least-privilege pattern — other jobs remain incapable of accidental writes.

### Pattern 2: On-Demand Tag Creation via `target_commitish`
**What:** Pass both `tag_name:` and `target_commitish:` to softprops/action-gh-release@v2. The action calls GitHub's REST `createRelease` API, which **creates the tag at the specified commitish if it doesn't exist yet**.
**When to use:** When triggering via `workflow_dispatch` (no tag in `github.ref`) and you want a tag created from scratch per-dispatch.
**Source code path** (from `src/main.ts` + `src/github.ts` in action v2):
```typescript
// When findTagFromReleases(tag) returns undefined (tag doesn't exist):
return await createRelease(tag, config, releaser, owner, repo, ...);
// Which calls:
await releaser.createRelease({
  owner, repo, tag_name, name, body, draft, prerelease,
  target_commitish,     // ← GitHub API creates the ref here
  discussion_category_name, generate_release_notes, make_latest, previous_tag_name,
});
```
**Why it works:** GitHub's Releases API creates the underlying Git tag when `target_commitish` is a valid commit SHA or branch name AND the `tag_name` doesn't already exist. Documented behavior — confirmed in action-gh-release#546 discussion. This is why a PAT is *not* required for a same-repo dispatch; the auto-provided `GITHUB_TOKEN` with `contents: write` scope is sufficient.

### Pattern 3: Re-Dispatch Clobber via `overwrite_files: true`
**What:** When re-dispatching with the same `(base_version, python_version, cuda_version)`, the wheel filename is deterministic. The action's default `overwrite_files: true` replaces the existing asset on the same tag rather than failing.
**When to use:** Per CONTEXT.md's fail-fix-retry design — latest green build wins for its exact filename slot. Different Python versions (e.g. cp310 vs cp311) produce different filenames, so they **accumulate** on the same tag rather than clobbering each other.
**Example:**
```yaml
- uses: softprops/action-gh-release@v2
  with:
    tag_name: ${{ steps.meta.outputs.tag }}
    files: dist/*.whl
    # overwrite_files defaults to true — explicit for documentation if desired:
    # overwrite_files: true
```

### Pattern 4: Grep-Assert with Character-Class Self-Match Guard
**What:** The lint-workflow job grows a step that greps for `needs: [build, smoke-test]` on the publish job. The pattern uses regex character classes (`[:]`, `[-]`) so the grep command's own source line doesn't self-match.
**When to use:** Extending Phase 1's existing ban-grep pattern. The Phase 1 ban-grep uses `allow[-]unsupported[-]compiler` for the same reason.
**Example:**
```yaml
# Source: Phase 1 ban-grep at lines 1649-1663 of build-wheels-cuda-windows.yaml
- name: 'Lint: assert publish gate on smoke-test'
  shell: bash
  run: |
    set -euo pipefail
    WORKFLOW='.github/workflows/build-wheels-cuda-windows.yaml'
    # Character-class escapes prevent self-match on this step's source line.
    # Invariant: exactly one match on the workflow file (the publish job's needs: array).
    MATCHES=$(grep -cE 'needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]' "$WORKFLOW" || true)
    if [[ "$MATCHES" -ne 1 ]]; then
      echo "::error file=$WORKFLOW::publish job's 'needs: [build, smoke-test]' gate is missing or duplicated (expected exactly 1 match, got $MATCHES)"
      echo "REMEDIATION: ensure the publish job declares 'needs: [build, smoke-test]' verbatim; smoke-test is a HARD publish gate (upstream #1543)."
      exit 1
    fi
    echo "OK: publish job's needs: [build, smoke-test] gate present (1 match)"
```
**Subtlety:** The pattern uses `\[\s*build\s*,\s*smoke[-]test\s*\]` — `\s*` tolerates YAML reformatting (e.g. `[build, smoke-test]` vs `[ build , smoke-test ]`) without weakening the assertion. The invariant flips from count=0 (ban-grep) to count=1 (required-presence-grep).

### Pattern 5: Inline Release Body via Heredoc + Random Delimiter
**What:** Multi-line release body content is built in one step and passed to the softprops action via GITHUB_OUTPUT. Use a random delimiter (timestamp + $RANDOM) not plain `EOF` to prevent accidental collision.
**When to use:** Any step that emits multi-line content through `$GITHUB_OUTPUT`.
**Example:**
```bash
# Source: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#multiline-strings
DELIM="BODY_$(date +%s)_$RANDOM"
{
  echo "content<<$DELIM"
  cat <<'INNER'
  multi-line
  release notes here
  INNER
  echo "$DELIM"
} >> "$GITHUB_OUTPUT"
```
**Why random delimiter:** The inner heredoc content (`<<'INNER'`) can contain literal `EOF` if a user-supplied input (e.g. a PR description) includes that string, which would truncate the output. Random delimiter is collision-safe.

### Anti-Patterns to Avoid

- **Workflow-level `permissions: contents: write`** — gives EVERY job write access, including lint-workflow and smoke-test. Use job-level scoping.
- **Relying on `${{ github.ref_name }}` for `tag_name`** — `workflow_dispatch` from `main` sets `github.ref_name=main`, which is not a valid tag name. Always compute the tag explicitly from the wheel filename's base version.
- **Using `actions/checkout@v4` on the publish job** — unnecessary; the wheel + metadata come from the artifact + prior-job outputs. Skipping checkout saves ~3 seconds per run and reduces attack surface.
- **Omitting `target_commitish`** — the default is the repo default branch, which is correct only if dispatched from `main`. Pinning to `${{ github.sha }}` is the reproducibility-safe choice.
- **Using `fail_on_unmatched_files: false`** — CONTEXT.md discretion says user didn't specify, but unmatched files means the download-artifact step silently failed. Prefer **`true`** (fail-loud) to catch broken Phase 2 contract early.
- **Repackaging the wheel before upload** — the bytes uploaded must be byte-identical to the smoke-tested artifact (CONTEXT.md specifics: "No repackaging, no re-signing, no metadata edits"). Do not run `wheel pack`, `twine`, or any repair tool between download and upload.
- **Plain `EOF` as heredoc delimiter in GITHUB_OUTPUT** — collision risk if body content contains literal `EOF`. Use random delimiter.
- **Printing the PAT or GITHUB_TOKEN to logs** — softprops masks the token, but ad-hoc `echo` or `curl -v` calls that include `Authorization:` headers will leak. No manual token manipulation needed — softprops handles it.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Release + asset upload | Custom `curl`/`gh` script hitting the Releases API | `softprops/action-gh-release@v2` | Locked by CONTEXT.md / PUB-02. Handles: tag creation via `target_commitish`, asset upload with retry, find-or-create release idempotency, clobber semantics, error messages, rate limit backoff. |
| Tag creation before release | Manual `git tag` + `git push` step | `target_commitish:` input to softprops | GitHub's Releases REST API auto-creates the tag; no separate `git tag` step needed. |
| Multi-line GITHUB_OUTPUT | String-replace `\n` escaping | Heredoc with random delimiter | Escaping `%`, `\n`, `\r` in bash is brittle; heredoc is the documented pattern. |
| Wheel hash for release body | Custom Python hashing script | `sha256sum` (coreutils, pre-installed on ubuntu-latest) | One-liner; no temp files. |
| Wheel size formatting | `bc` for floating-point | `awk` or bash arithmetic | `awk` ships on every ubuntu runner; `bc` sometimes requires install. |
| Release body templating | Jinja2 / ERB / external template file | Inline bash heredoc + `${{ }}` expression substitution | CONTEXT.md's "No shell scripts outside the workflow" constraint rules out external templates. Action expression substitution covers all variable interpolation. |
| README table-of-contents generator | `markdown-toc` or similar tool | Hand-written section headers | The new README is ~100 lines; TOC is overkill. |

**Key insight:** Phase 4's publish job is almost entirely **action glue**. The only shell logic is metadata extraction (wheel filename parsing, hash, size) and release body assembly. CONTEXT.md's constraint (no shell scripts outside the workflow) is naturally satisfied because softprops does all the heavy lifting.

## Common Pitfalls

### Pitfall 1: Tag Not Created (Missing `target_commitish`)
**What goes wrong:** softprops attempts to create a release for a tag that doesn't exist, with no `target_commitish`. GitHub API returns 422 "tag_name is not a valid tag" or "Release.target_commitish is invalid."
**Why it happens:** `workflow_dispatch` sets `github.ref=refs/heads/main`, not `refs/tags/...`. The action's default `target_commitish` is the repo default branch, which *usually* works — but if dispatched from a non-`main` branch, the action creates the tag on a branch that may not contain the build's commit. Reproducibility is broken.
**How to avoid:** Always set `target_commitish: ${{ github.sha }}` explicitly. The tag then pins to the exact commit of the build.
**Warning signs:** `422 Validation Failed` in the softprops step log; release exists but tag points to wrong commit; `git describe` on the tag returns unexpected commit.

### Pitfall 2: Job-Level Permissions Shadowing
**What goes wrong:** Workflow-level `permissions:` is set (e.g. inherited from a template) and the publish job's job-level `permissions:` block is *narrower* than what's needed. GitHub Actions applies job-level as a strict override, not an additive scope — anything not listed becomes `none`.
**Why it happens:** Developer assumes job-level adds to workflow-level. Actually, if job-level specifies any permission, all unspecified ones are `none`.
**How to avoid:** List all required permissions at the job level. For publish, `contents: write` is enough. If the job ever needs to read packages or issues, list those too.
**Warning signs:** "Resource not accessible by integration" errors at runtime; 403 on API calls that worked in previous workflows.

### Pitfall 3: Heredoc Delimiter Collision in GITHUB_OUTPUT
**What goes wrong:** Release body content contains the literal string `EOF` (e.g. from a user-provided input). The heredoc terminates early; the output is truncated; downstream step sees a partial body.
**Why it happens:** Plain `EOF` is the idiomatic heredoc delimiter but is not collision-safe for arbitrary content.
**How to avoid:** Use a random delimiter: `DELIM="BODY_$(date +%s)_$RANDOM"`. Even if the body content contains `EOF`, it won't match the random delimiter.
**Warning signs:** Release body shows up truncated or garbled; `$GITHUB_OUTPUT` file contains an unterminated heredoc.

### Pitfall 4: Artifact Name Mismatch Between download-artifact and upload-artifact
**What goes wrong:** Publish job's `actions/download-artifact@v4` uses `name: cuda-wheels` (plural), but build job uses `name: cuda-wheel` (singular). Download produces an empty directory; glob matches nothing; `fail_on_unmatched_files: true` catches it (if set).
**Why it happens:** Typo between jobs; Phase 2 contract is easy to miss.
**How to avoid:** The contract is `name: cuda-wheel` (singular). Verify by grepping `.github/workflows/build-wheels-cuda-windows.yaml` for `name: cuda-wheel` and counting matches (upload + download should both match). Use `fail_on_unmatched_files: true` as the loud safety net.
**Warning signs:** "No files were found with the provided path: dist/*.whl"; publish job exits 0 with no asset uploaded.

### Pitfall 5: Wheel Version Cannot Be Parsed from Filename
**What goes wrong:** The metadata step's `sed -E 's/^llama_cpp_python-([0-9.]+)\+.*/\1/'` regex fails if Phase 2 ever drifts from the locked filename scheme (e.g. `abi3` instead of `cp311`, or no `+cu126.ll<sha>` local-version segment).
**Why it happens:** Silent drift in Phase 2's `SETUPTOOLS_SCM_PRETEND_VERSION` logic; downstream phase assumes a format that changed.
**How to avoid:** After the `sed` extraction, assert the tag looks sane (`[[ "$TAG" =~ ^v[0-9]+\.[0-9]+\.[0-9]+-cu126-win$ ]]`) and fail loudly with remediation if not. Phase 2 Plan 02 already asserts the wheel tag regex (`BLD-10`) — this is belt-and-suspenders.
**Warning signs:** Tag is empty or `v-cu126-win`; softprops step fails with "tag_name is required."

### Pitfall 6: README Overwrite Breaks Existing Documentation Links
**What goes wrong:** Anything referencing `README.md` sections that existed in the upstream (824 lines of API docs) breaks silently — docs.readthedocs.io redirects, PyPI README, external wikis.
**Why it happens:** README overwrite is a big semantic change for anyone who landed from upstream.
**How to avoid:** The new README's **Attribution** section must prominently link to `README.upstream.md`. The `README.upstream.md` file is preserved byte-for-byte (no edits per CONTEXT.md). Internal relative links in upstream README still work because the file is preserved.
**Warning signs:** User reports 404 on old README anchors; external wiki links break. (Low-impact for a solo fork with no PyPI publish, but worth noting.)

### Pitfall 7: Softprops Permissions on First-Run Tag Protection
**What goes wrong:** If the repo has **tag protection rules** (e.g. protecting `v*` tags), softprops' tag-creation call returns 403 even with `contents: write`.
**Why it happens:** Tag protection is a separate RBAC layer on top of `contents:` permission.
**How to avoid:** Check repo Settings → Rules → Rulesets for any tag-protection rule. For a solo fork, this is usually not configured. If it is, grant the workflow's service principal bypass access, or use a PAT.
**Warning signs:** `403 Resource not accessible by integration` only on the Release step (Download/Upload artifact works fine).

### Pitfall 8: Re-Dispatch with Different `cuda_version` Uploads to Wrong Tag
**What goes wrong:** Someone dispatches with `cuda_version=12.4.1` (hypothetically, after a matrix expansion), but the tag scheme is hardcoded to `cu126`. The wheel gets uploaded to the `v<ver>-cu126-win` tag despite being a cu124 wheel.
**Why it happens:** The current tag scheme hardcodes `cu126` in the metadata step. Works fine for v1 (single CUDA version) but breaks if v2 ever adds `cuda_version` matrix.
**How to avoid:** Derive the `cu<tag>` component from `inputs.cuda_version` (e.g. `12.6.3` → `cu126`). Even in v1, making the tag data-driven prevents a latent bug:
```bash
CU_MAJORMINOR=$(echo "${{ inputs.cuda_version }}" | awk -F. '{printf "cu%s%s", $1, $2}')
TAG="v${BASE_VER}-${CU_MAJORMINOR}-win"
```
**Warning signs:** In v2, cu124 and cu126 wheels land on the same Release tag; consumers install the wrong one.

## Code Examples

Verified patterns from official sources and existing workflow code:

### Softprops v2 Minimal Invocation (from action README, v2 tag)

```yaml
# Source: https://github.com/softprops/action-gh-release/blob/v2/README.md
- uses: softprops/action-gh-release@v2
  with:
    tag_name: v1.0.0
    files: |
      dist/*.whl
      dist/*.tar.gz
    # Token defaults to ${{ github.token }} — no need to pass explicitly.
```

### GitHub Output Multi-Line with Random Delimiter (from GitHub docs)

```bash
# Source: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#multiline-strings
DELIM="RELEASE_BODY_$(date +%s)_$RANDOM"
{
  echo "content<<$DELIM"
  cat <<'INNER'
  line one
  line two with ${{ ... }} substitutions handled by Actions expression layer
  INNER
  echo "$DELIM"
} >> "$GITHUB_OUTPUT"
```

### Job-Level Permissions for Release Upload (from GitHub docs)

```yaml
# Source: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write     # Sufficient for softprops@v2; all other permissions implicitly `none`.
    steps:
      - uses: softprops/action-gh-release@v2
        with:
          files: dist/*.whl
```

### Grep-Assert with Self-Match Guard (from Phase 1, lines 1649-1663)

```bash
# Source: C:\claude_checkouts\llama-cpp-python\.github\workflows\build-wheels-cuda-windows.yaml:1649
set -euo pipefail
WORKFLOW='.github/workflows/build-wheels-cuda-windows.yaml'
if grep -nHE 'allow[-]unsupported[-]compiler' "$WORKFLOW"; then
  echo "::error file=$WORKFLOW::the banned CMake flag is present (upstream #1543)"
  exit 1
fi
echo "OK: no banned compiler-bypass workaround flag in $WORKFLOW"
```

### Phase 4 Extension — Gate Assertion (new)

```bash
# Pattern: presence-grep (count=1) instead of absence-grep (count=0).
set -euo pipefail
WORKFLOW='.github/workflows/build-wheels-cuda-windows.yaml'
MATCHES=$(grep -cE 'needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]' "$WORKFLOW" || true)
if [[ "$MATCHES" -ne 1 ]]; then
  echo "::error file=$WORKFLOW::publish gate regression (expected 1 match, got $MATCHES)"
  echo "REMEDIATION: restore 'needs: [build, smoke-test]' on the publish job. See upstream #1543 for rationale."
  exit 1
fi
echo "OK: publish gate present (1 match)"
```

### Release Body Forensics Table (drafted from CONTEXT.md decisions)

```markdown
Windows CUDA 12.6 wheel, built 2026-04-19 against llama.cpp @ 227ed28e128e.

| Property | Value |
|---|---|
| llama.cpp SHA | `227ed28e128e` |
| Base version | `0.3.20` |
| CUDA version | `12.6.3` |
| MSVC toolset | `14.44` |
| Python | `3.11` |
| Wheel size | `375123456` bytes (357.8 MiB) |
| Wheel SHA-256 | `a3f2...ef01` |
| Build date (UTC) | `2026-04-19` |
| Dispatch run | https://github.com/ThongvanAlexis/llama-cpp-python/actions/runs/12345678 |
| Smoke-test | green (publish gated on `needs: [build, smoke-test]`) |
```

### Inter-Job Output Passing (build job → publish job)

The publish job needs `llama_sha` and `selected_toolset` from prior jobs. This requires build job outputs exposed at the job level:

```yaml
# In the 'build:' job header — add this IF NOT ALREADY PRESENT:
build:
  runs-on: windows-2022
  outputs:
    llama_sha: ${{ steps.assert-submodule.outputs.llama_sha }}
    selected_toolset: ${{ steps.probe-msvc.outputs.selected_toolset }}
    selected_full_version: ${{ steps.probe-msvc.outputs.selected_full_version }}
  steps:
    ...

# In the publish job — consume via needs.build.outputs:
publish:
  needs: [build, smoke-test]
  steps:
    - run: echo "llama.cpp SHA: ${{ needs.build.outputs.llama_sha }}"
```

**Verification required during planning:** `grep -nE "^[[:space:]]+outputs:" build-wheels-cuda-windows.yaml` — if the build job has no top-level `outputs:` block, the publish job cannot reach `steps.*.outputs.*` from the build job. Currently (Phase 2 end-state): the workflow has step-level outputs on probe-msvc and assert-submodule but **no job-level outputs** — Phase 4 must add them.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `actions/create-release@v1` (archived) | `softprops/action-gh-release@v2` (or v3) | 2020 — actions/create-release archived | Use softprops exclusively. v1 is EOL; v2 is stable and maintained alongside v3. |
| Node 20 action runtime (v2) | Node 24 action runtime (v3, released 2026-04-12) | 2026-04-12 | v2 stays Node-20-compatible; safe for current images. No need to migrate to v3 for Phase 4. |
| `gh release create` via shell | `softprops/action-gh-release@v2` | 2022+ | Softprops handles tag creation, idempotency, retry. `gh` CLI is a viable fallback but more shell boilerplate. |
| PAT for Release creation | Auto-provided `GITHUB_TOKEN` with `contents: write` | 2021+ (GitHub Actions enhancement) | Same-repo release creation needs no PAT. Cross-repo does. |
| Plain `EOF` heredoc for multi-line output | Random delimiter (`EOF_$(date +%s)_$RANDOM`) | 2023+ best practice | Prevents collision when content contains literal `EOF`. |
| Workflow-level `permissions: write-all` | Job-level `permissions: contents: write` (least privilege) | 2022+ (restricted default tokens) | Default GITHUB_TOKEN is read-only on new repos (November 2021 change); explicit write must be granted per-job. |

**Deprecated/outdated:**
- **`actions/create-release@v1`** — archived Dec 2021. Replaced by softprops/action-gh-release.
- **`actions/upload-release-asset@v1`** — archived Dec 2021. Same replacement.
- **PEP 503 index on gh-pages (original Phase 4 plan)** — dropped for THIS FORK in v1 per CONTEXT.md 2026-04-19. Still valid approach for larger forks; revisit in v2.

## Open Questions

1. **Build job outputs block existence**
   - What we know: `probe-msvc` and `assert-submodule` produce step outputs, but the build job header may lack a top-level `outputs:` block exposing them.
   - What's unclear: whether Phase 2's final workflow includes job-level outputs. Need to grep the existing file during planning.
   - Recommendation: First task of Plan 04-01 is to verify with `grep -nE '^    outputs:' .github/workflows/build-wheels-cuda-windows.yaml`. If missing, add the outputs block to the build job as a pre-requisite step (tiny change, same file).

2. **`nvidia-smi` example output snippet in README**
   - What we know: CONTEXT.md marks this as Claude's discretion.
   - What's unclear: user preference — polish vs terseness.
   - Recommendation: include a brief formatted code block (3-4 lines) showing the relevant fields of `nvidia-smi` output so consumers can self-verify without cross-referencing. Maximum polish for minimum words.

3. **Exact driver version in README (527.41 vs 561.17 vs CONTEXT.md's "560.x")**
   - What we know: NVIDIA's official docs give **≥ 527.41** for any CUDA 12.x runtime on Windows (minor version compat) and **≥ 561.17** for installing the CUDA 12.6.3 toolkit.
   - What's unclear: CONTEXT.md specifies "560.x" as a rough figure.
   - Recommendation: use **≥ 561.17** in the README as the safe/canonical number — it's the driver that ships with CUDA 12.6.3, matches what `nvidia-smi` will report on an up-to-date machine, and exceeds the minor-version-compat floor. This is a tiny correction to CONTEXT.md's "560.x"; flag it to the user for confirmation but don't block. Pragmatically: any consumer running recent gaming/workstation drivers (2024+) will exceed both.

4. **Whether to use `generate_release_notes: true`**
   - What we know: softprops supports auto-generating release notes from commit messages since the last tag. Default is false.
   - What's unclear: whether the auto-generated commit log adds signal or noise to the forensics-focused body CONTEXT.md designed.
   - Recommendation: leave `generate_release_notes: false` (default). The curated forensics table is the whole point of the release body; auto-generated commit lists would dilute it. Revisit in v2 if consumers request a changelog.

5. **Plan split: 1 vs 2 plans**
   - What we know: ROADMAP sketched 2 plans (04-01 publish job + 04-02 README / REQUIREMENTS amendments). CONTEXT.md leaves this to planner.
   - What's unclear: whether README + REQUIREMENTS changes warrant a separate plan.
   - Recommendation: **single combined plan (04-01)**. Rationale: (a) the scope reduction means the publish job is much smaller than originally envisioned; (b) README + publish job must land together or the release body references a URL path (`github.com/.../releases`) the README doesn't yet explain; (c) REQUIREMENTS.md amendments are a trivial markdown edit — under 10 lines of diff; (d) all four files (workflow YAML, README.md, README.upstream.md rename, REQUIREMENTS.md) touch orthogonal file paths with no task-level dependencies.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | GitHub Actions self-verification (workflow lint + dispatch result). No pytest needed for Phase 4 — no Python code is added. |
| Config file | `.github/workflows/build-wheels-cuda-windows.yaml` (existing) |
| Quick run command | `bash <(curl -sSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash) && ./actionlint -color .github/workflows/build-wheels-cuda-windows.yaml` (already wired in lint-workflow job) |
| Full suite command | `gh workflow run build-wheels-cuda-windows.yaml -f python_version=3.11 -f cuda_version=12.6.3` then `gh run watch` (manual dispatch verification) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| PUB-01 | Publish job runs on ubuntu-latest | static (yaml-grep) | `grep -cE '^  publish:' .github/workflows/build-wheels-cuda-windows.yaml` must equal 1; then `grep -A2 '^  publish:' ... \| grep -cE 'runs-on:[[:space:]]*ubuntu-latest'` must equal 1 | ✅ (workflow file exists) |
| PUB-02 | Wheel uploaded via softprops/action-gh-release@v2 | static (yaml-grep) + dispatch | `grep -cE 'softprops/action-gh-release@v2' .github/workflows/build-wheels-cuda-windows.yaml` must equal 1 | ✅ |
| DOC-01 | README has Install (Windows CUDA) section | static (md-grep) | `grep -cE '^## Install.*[Ww]indows.*CUDA' README.md` must equal 1; `grep -cE 'https://github\.com/ThongvanAlexis/llama-cpp-python/releases' README.md` must equal 1 | ❌ (README not yet rewritten — created by this phase) |
| DOC-02 | README notes NVIDIA driver floor | static (md-grep) | `grep -cE '(nvidia-smi\|561\.17\|driver.*\b5[0-9][0-9]\b)' README.md` must be ≥ 1 | ❌ |
| DOC-05 | Release body records llama.cpp SHA | dispatch | After successful publish dispatch: `gh release view v<ver>-cu126-win --json body -q .body \| grep -cE 'llama\.cpp.*SHA'` must equal ≥ 1 | — (produced by running workflow) |
| Gate assertion | lint-workflow fails if `needs: [build, smoke-test]` is missing | unit (static grep) | Remove the `needs:` line locally → `actionlint` + the new grep step should both fail | ✅ (lint-workflow job exists at line 1642) |

### Sampling Rate
- **Per task commit:** `actionlint .github/workflows/build-wheels-cuda-windows.yaml` (static YAML lint) + `grep`-based invariants listed above.
- **Per wave merge:** full workflow_dispatch on a test branch (cannot be cheaper — the publish job needs the prior-jobs' artifacts and outputs, so the whole workflow must run). Expected cost: ~2-3 hours if caches are warm from recent Phase 3 dispatches.
- **Phase gate:** one full dispatch that produces a green publish job, a real GitHub Release at the expected tag, an uploaded wheel asset, and a body containing the llama.cpp SHA and forensics table. Verify by running `gh release view <tag>` and `pip install <asset-url>` in a fresh venv on the dev Windows host.

### Wave 0 Gaps
- [ ] `README.upstream.md` — rename target for the 824-line upstream README. File does not yet exist; created by `git mv README.md README.upstream.md`.
- [ ] New `README.md` — fork-focused content per CONTEXT.md structure. Does not yet exist in its new form.
- [ ] Build job `outputs:` block — probably missing from the existing workflow. Verify with `grep -nE '^    outputs:' .github/workflows/build-wheels-cuda-windows.yaml`; add before publish job if absent.
- [ ] No Python test framework gaps — Phase 4 adds no Python code.
- [ ] No new framework installs needed — `actionlint` is already downloaded ad-hoc in the lint-workflow job (line 1669).

## Sources

### Primary (HIGH confidence)
- **softprops/action-gh-release action.yml (v2 branch)** — `https://raw.githubusercontent.com/softprops/action-gh-release/v2/action.yml` — authoritative input list, defaults
- **softprops/action-gh-release v2 README** — `https://github.com/softprops/action-gh-release/blob/v2/README.md` — full input/output spec, permissions requirement, v2 vs v3 distinction
- **softprops/action-gh-release src/github.ts** — `https://github.com/softprops/action-gh-release/blob/v2/src/github.ts` — code path confirming `createRelease` is called with both `tag_name` and `target_commitish`, enabling on-demand tag creation
- **GitHub Actions — Workflow syntax (permissions)** — `https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#permissions` — job-level override semantics ("unspecified permissions are set to `none`")
- **GitHub Actions — Controlling permissions for GITHUB_TOKEN** — `https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token` — minimum scope for release creation
- **NVIDIA CUDA 12.6.3 Release Notes** — `https://docs.nvidia.com/cuda/archive/12.6.3/cuda-toolkit-release-notes/index.html` — Windows driver floor (561.17) for CUDA 12.6.3
- **NVIDIA CUDA Minor Version Compatibility** — `https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html` — CUDA 12.x compat floor (>=525 Linux, 527.41 Windows equivalent)
- **Existing workflow file** — `C:\claude_checkouts\llama-cpp-python\.github\workflows\build-wheels-cuda-windows.yaml` — Phase 1-3 patterns (ban-grep self-match guard at line 1649-1663, lint-workflow job layout at line 1642, smoke-test green forensics pattern at line 1503-1536)
- **Upstream reference usage** — `.github/workflows/build-and-release.yaml:202` — existing `softprops/action-gh-release@v2` invocation in the same repo (upstream's path, minimal example)

### Secondary (MEDIUM confidence)
- **GitHub Actions — Workflow commands (multiline strings)** — `https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#multiline-strings` — heredoc pattern for GITHUB_OUTPUT
- **action-gh-release Issue #339** (tag creation requirement discussion) — confirms the target_commitish auto-tag-creation path
- **action-gh-release Issue #546** (`target_commitish` with `repository`) — concrete example of `target_commitish: ${{ github.sha }}` usage
- **NVIDIA Data Center Driver 525.60.13/527.41 Release Notes** — confirms 527.41 is the Windows counterpart of Linux 525.60.13 (the CUDA 12.x floor driver)

### Tertiary (LOW confidence)
- **DeepWiki softprops/action-gh-release — Release Process Workflow** — `https://deepwiki.com/softprops/action-gh-release/4.2-release-process-workflow` — claimed "the action does NOT create tags", which **contradicts** the action.yml source and issue #546. Disregarded in favor of HIGH-confidence sources. Flagged for reader.

## Metadata

**Confidence breakdown:**
- Standard stack (softprops v2, download-artifact v4): HIGH — action.yml and README quoted verbatim; versions pinned.
- Architecture patterns (job-level permissions, on-demand tag creation, grep-assert with self-match guard): HIGH — GitHub official docs + action source code + existing Phase 1 pattern already in-repo.
- Pitfalls: HIGH for 1, 2, 3, 4, 7 (documented behavior). MEDIUM for 5, 6, 8 (extrapolated from code structure but not yet empirically tested).
- DOC-02 driver version: HIGH (NVIDIA official release notes); note that CONTEXT.md's "560.x" was an approximation — corrected to 561.17 (toolkit) or 527.41 (runtime-only). Planner should pick 561.17 for README safety.
- Release body + README content: HIGH for structure (CONTEXT.md lockdowns are crisp); MEDIUM for exact wording (Claude's discretion per CONTEXT.md).

**Research date:** 2026-04-19
**Valid until:** 2026-05-19 (30 days — softprops v2 is stable; NVIDIA driver table changes slowly; GitHub Actions permission model is stable). If v3 migration becomes necessary (e.g. v2 security EOL), re-research at that point.
