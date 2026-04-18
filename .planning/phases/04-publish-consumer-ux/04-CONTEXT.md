# Phase 4: Publish & Consumer UX - Context

**Gathered:** 2026-04-19
**Status:** Ready for planning

<domain>
## Phase Boundary

On green smoke-test, upload the built wheel as an asset on a GitHub Release (authoritative storage), and update the repository README so a consumer can reliably download and `pip install` the wheel on Windows + CUDA 12.6. Publish job runs on `ubuntu-latest`, declares `needs: [build, smoke-test]` so a red smoke test blocks publish, and is guarded by a grep-assert in `lint-workflow` against future edits silently removing that gate. Release body embeds the llama.cpp submodule SHA (build-time forensics).

**SCOPE REDUCTION — PEP 503 / gh-pages dropped on 2026-04-19.** Original phase scope included a `gh-pages` PEP 503 index with SHA-256 fragments, `data-requires-python` attrs, `peaceiris/actions-gh-pages@v4` + `keep_files: true`, `concurrency: gh-pages-publish`, and a post-publish Fastly probe (PUB-03..10, DOC-03). All dropped. Consumers install by manually downloading the wheel from the Releases page and running `pip install path/to/wheel.whl`. REQUIREMENTS.md needs an amendment to move PUB-03..PUB-10 and DOC-03 to "Out of Scope"; DOC-01's canonical command is reinterpreted as manual download + pip install.

Phase 4 is the last phase of v1 — there are no downstream phases reading this CONTEXT.md inside the current milestone.

</domain>

<decisions>
## Implementation Decisions

### Release tag & asset scheme
- **Tag format**: `v<base_version>-cu126-win` (e.g. `v0.3.20-cu126-win`). Namespaces Windows so future Linux/Mac wheels can coexist under different tag suffixes. One Release per `(base_version, cu-tag, os)` tuple; assets accumulate across Python versions on the same tag.
- **Asset upload**: `softprops/action-gh-release@v2` (PUB-02). Release is the sole storage location — no gh-pages duplication.
- **Re-dispatch behavior**: clobber. Same wheel filename → overwrite existing asset on the same tag. Latest green build wins for its exact filename slot. Fits the fail-fix-retry CI loop without producing history clutter.
- **Wheel filename**: keep Phase 2's `+cu126.ll<sha12>` local-version segment (`llama_cpp_python-0.3.20+cu126.ll227ed28e128e-cp311-cp311-win_amd64.whl`). No rework of Phase 2's `SETUPTOOLS_SCM_PRETEND_VERSION` logic. SHA-in-filename proves the embedded llama.cpp commit at a glance even when the wheel is archived without the Release context.
- **First-publish bootstrap**: softprops auto-creates the Release on first push of a tag. No manual pre-creation needed.

### Publish job layout
- **Runner**: `ubuntu-latest` (PUB-01). Cheap minutes; no CUDA or MSVC needed for a Release asset upload.
- **Dependencies**: `needs: [build, smoke-test]` — hard gate. Publish is structurally incapable of running if smoke-test is red.
- **Permissions**: `contents: write` on the publish job only (enables Release asset upload with the auto-provided `GITHUB_TOKEN`; no PAT needed for same-repo pushes).
- **Concurrency**: NONE. `workflow_dispatch` requires Write access, so on a public-but-solo fork only maintainers can trigger parallel runs. User will self-coordinate; accept the near-zero risk of parallel same-input dispatches.
- **Steps**: download `cuda-wheel` artifact (Phase 2 contract) → read wheel filename + compute metadata → construct Release body → softprops upload with `fail_on_unmatched_files: false`. Each step has explicit `timeout-minutes`.

### Gate assertion (deferred from Phase 3)
- **Location**: extend the existing `lint-workflow` ubuntu job with a new step alongside the ban-grep and actionlint. No new job; no new runner minutes.
- **Check**: grep the workflow YAML for the literal `needs: [build, smoke-test]` line on the publish job; fail with a remediation hint if missing.
- **Pattern**: same character-class-escape trick used by Phase 1's ban-grep (`needs[:].*build.*smoke[-]test` or similar) to avoid self-match on the lint step's own source line.

### Release body (DOC-05 + forensics)
- **Format**: narrative intro + full forensics table.
- **Intro line**: `Windows CUDA 12.6 wheel, built YYYY-MM-DD against llama.cpp @ <sha12>.`
- **Table columns**: llama.cpp SHA (12-char, required by DOC-05), base version, CUDA version, MSVC toolset selected, Python version, wheel size (bytes / MiB), build date (UTC), dispatch run URL, smoke-test exit code.
- **Generation**: assembled inline in the publish job using outputs from prior jobs (`assert-submodule.outputs.llama_sha`, `probe-msvc.outputs.selected_toolset`, wheel metadata, `github.run_id`, etc.). No external template file.

### README strategy
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

### REQUIREMENTS.md amendments (Plan 4-02 will apply these)
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

</decisions>

<specifics>
## Specific Ideas

- User quote on README intent: *"this fork just adds a CI on top of the original llama-cpp in order to build a windows wheel with prepackage cuda dlls (no cuda install needed) then explain how we build it"* — capture this framing in the intro paragraph.
- User quote on concurrency: *"if other people than me can run the jobs (repo is public, can other people run jobs?) then yes, otherwise no, I'll be careful"* — resolved: `workflow_dispatch` needs Write access, solo maintainer, skip concurrency.
- User pivoted away from gh-pages mid-discussion on 2026-04-19 — capture the pivot in PROJECT.md Key Decisions so the rationale survives future "why not gh-pages?" questions.
- Wheel bytes on Release = the exact same bytes uploaded as the `cuda-wheel` artifact in Phase 2. No repackaging, no re-signing, no metadata edits in the publish job.
- Release body's `smoke-test exit code` column is load-bearing for the fork's core value proposition — it's the visible proof the wheel passed the DLL-load gate, not just the build.
- The narrative intro line ("Windows CUDA 12.6 wheel, built YYYY-MM-DD against llama.cpp @ \<sha12\>") should be the first line of the Release body so it appears in GitHub's release list previews.

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- **`cuda-wheel` artifact** (Phase 2 contract): published by the build job, consumed by both smoke-test and publish. `actions/download-artifact@v4` with `name: cuda-wheel` is the standard consumer pattern.
- **`assert-submodule` step outputs** (Phase 2): `llama_sha` is available in-workflow — reuse for Release body without re-deriving.
- **`probe-msvc` step outputs** (Phase 1): `selected_toolset`, `selected_full_version`, `msc_ver_cap` — reuse for Release body forensics table.
- **`lint-workflow` job** (Phase 1): ubuntu-latest, already contains ban-grep + actionlint. Adding one more grep step is near-free. Pinned `@v4` action versions.
- **`softprops/action-gh-release@v2`**: already used in upstream's `build-and-release.yaml` (line 202); locked by PUB-02.
- **Upstream `scripts/releases-to-pep-503.sh` + `scripts/get-releases.sh`**: originally relevant to PUB-04 — now irrelevant with gh-pages dropped. Scripts unchanged (upstream-owned, not our concern).
- **Upstream `README.md` (824 lines)**: becomes `README.upstream.md` after the overwrite. No content edits to the preserved file.

### Established Patterns
- `shell: pwsh` on Windows steps; `shell: bash` on ubuntu steps (publish job is ubuntu → bash).
- `set -euo pipefail` prelude for all bash blocks.
- Action pins: `@v4`/`@v5` semantic tags throughout.
- Hard-fail with remediation hint on assertion failures (no warn-and-continue anywhere).
- `$GITHUB_STEP_SUMMARY` forensics on green; remediation diag on red (publish job should emit a green summary on successful asset upload mirroring this pattern).
- Every step gets an explicit `timeout-minutes`.

### Integration Points
- `needs: [build, smoke-test]` on publish job — the structural publish gate.
- `lint-workflow` job: add grep-assert step for the `needs:` array.
- README.md at repo root — replaced; upstream content preserved at README.upstream.md.
- No CMakeLists.txt / pyproject.toml / llama_cpp/ changes in this phase.

</code_context>

<deferred>
## Deferred Ideas

- **PEP 503 index on gh-pages** — original Phase 4 scope. Dropped 2026-04-19. Rationale: maintenance cost of gh-pages branch + index regeneration + Fastly probe + `--extra-index-url` UX didn't justify the complexity for a solo-maintainer fork. If the fork grows a wider user base or CI matrix, revisit as a standalone phase in v2. When revisited: the requirement set is PUB-03..PUB-10 + DOC-03; the index generator must regenerate from dir listing (not GH API) with SHA-256 fragments and `data-requires-python`; bootstrap first-time gh-pages branch; post-publish Fastly probe with 15×60s timeout.
- **Auto-trigger on tag push / Release publish** — `AUT-01` / `AUT-02`; v2 scope.
- **Sigstore / PEP 740 attestations on uploaded wheels** — `QA-01`; v2 scope.
- **CHANGELOG.md stub** — surfaced during discussion, not decided. Low cost to add a skeleton ("Windows CUDA wheel builds start with v0.3.20-cu126-win"), but not required by any v1 requirement. Planner's call.
- **`nvidia-smi` example output screenshot/snippet in README** — polish, not a requirement. Planner may include a formatted code block showing expected `nvidia-smi` output to make the prereqs step self-verifiable.
- **Multi-Python matrix / multi-CUDA matrix wheels on the same Release tag** — works out of the box with the chosen `v<ver>-cu126-win` scheme (assets accumulate). Not a Phase 4 task to exercise; falls out as a v2 matrix expansion (`MX-01`, `MX-02`).
- **Repository-wide ban-grep as always-on PR check** — still deferred from Phase 1.

</deferred>

---

*Phase: 04-publish-consumer-ux*
*Context gathered: 2026-04-19*
