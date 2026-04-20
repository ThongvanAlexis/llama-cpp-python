---
phase: 04-publish-consumer-ux
plan: 02
subsystem: infra
tags: [github-actions, workflow-dispatch, softprops-action-gh-release, cuda, windows, release-forensics]

# Dependency graph
requires:
  - phase: 04-publish-consumer-ux
    provides: "Plan 04-01 landed the publish job, lint-gate presence-grep, fork-focused README, and REQUIREMENTS/PROJECT scope-reduction amendments. Plan 04-02 proves that scaffolding works end-to-end on a real dispatch."
provides:
  - "Green end-to-end workflow_dispatch run (ID 24623672816) on target commit 973aeff, all four jobs conclusion=success (build + smoke-test + publish + lint-workflow)"
  - "Live GitHub Release at tag v0.3.20-cu126-win with the Windows CUDA wheel as an asset (byte-identical upload verified: asset SHA-256 matches publish-job forensics)"
  - "DOC-05 satisfied on a real rendered release body: 12-char llama.cpp SHA (227ed28e128e) appears in both the narrative intro and the forensics table; full forensics table renders correctly in the github.com UI"
  - "Dispatch forensics archived: 04-02-run-meta.json (run metadata + per-job timings), 04-02-release.json (release body + asset metadata), 04-02-publish-log.txt (publish-job step-by-step log)"
  - "CUDA-runtime-bundling disclosure pattern on release body (both workflow template and live release patched so future dispatches inherit it)"
affects: [phase-verification, v1-milestone-close, consumer-UX-readiness]

tech-stack:
  added: []
  patterns:
    - "Dispatch-verification plans record runtime forensics (gh CLI JSON + logs) as artifacts under runtime_artifacts in plan frontmatter, not as files_modified — keeps frontmatter-schema checks that count source edits honest."
    - "Release body CUDA-bundling disclosure: clarify up-front that the wheel ships the CUDA runtime DLLs and a local CUDA Toolkit install is not required — lowers the install-barrier ask from 'install the Toolkit + matching driver' to 'have a driver'."

key-files:
  created:
    - ".planning/phases/04-publish-consumer-ux/04-02-run-meta.json"
    - ".planning/phases/04-publish-consumer-ux/04-02-release.json"
    - ".planning/phases/04-publish-consumer-ux/04-02-publish-log.txt"
    - ".planning/phases/04-publish-consumer-ux/04-02-SUMMARY.md"
  modified:
    - ".github/workflows/build-wheels-cuda-windows.yaml (SC2012 wheel-detection fix in commit 973aeff; CUDA-bundling disclosure in release-body template in commit d52a043)"
    - ".gitignore (ignore local actionlint binary + .claude agent config; commit e58d938)"

key-decisions:
  - "Dispatched against branch windows-build (not main) per user direction; both branches pushed to origin in sync after the run. Target commit on the Release is 973aeff (the SC2012 fix)."
  - "Remediated initial SC2012 shellcheck failure via Option A: nullglob array for wheel detection (instead of ls | grep). One-line change, no schema churn, no re-plan needed."
  - "Surfaced content gap during Task 2 visual inspection: the release body did not disclose that the wheel bundles the CUDA runtime. Patched via two-path approach — (a) gh release edit on the live v0.3.20-cu126-win body, (b) workflow template updated so the next dispatch inherits the disclosure. Both committed (d52a043 + 1d1876b)."

patterns-established:
  - "Release body disclosure pattern: front-load consumer-facing runtime/driver/toolkit prerequisites in the first 2-3 lines of the body, before the numbered Install steps, so readers scanning on github.com see the 'what do I need?' answer without scrolling."
  - "Two-path content-patch pattern: when a release-body content gap is found after publish, edit the live release (gh release edit, visible to consumers now) AND the workflow template (so future dispatches don't regress). Commit both in the same topical series so the SUMMARY can cite them as one remediation."

requirements-completed: [PUB-01, PUB-02, DOC-05]

# Metrics
duration: ~8h elapsed (dispatch wall-time 2h 1m for the workflow itself; remainder = pre-flight, remediation loop for SC2012, checkpoint wait, CUDA-disclosure remediation loop)
completed: 2026-04-19
---

# Phase 04 Plan 02: End-to-end Dispatch Verification Summary

**Live green workflow_dispatch publishes v0.3.20-cu126-win with byte-identical wheel (SHA-256 47649ad0...adbeed8), 12-char llama.cpp SHA (227ed28e128e) rendered in both narrative and forensics table, and CUDA-runtime-bundling disclosure added to both live release and workflow template.**

## Performance

- **Duration:** ~8h elapsed (includes 2h 1m dispatch wall-time, SC2012 remediation loop, checkpoint wait, CUDA-disclosure remediation loop)
- **Started:** 2026-04-19 (local pre-flight)
- **Dispatch launched:** 2026-04-19T07:20:24Z (release createdAt)
- **Dispatch green:** 2026-04-19T09:21:05Z (publish job completedAt)
- **User approval:** 2026-04-19 after Task 2 visual inspection (`approved`)
- **Tasks:** 2 planned + 1 pre-flight remediation = 3 execution segments
- **Files modified (source):** 2 (workflow YAML + .gitignore)
- **Runtime artifacts captured:** 3 (run-meta.json, release.json, publish-log.txt)

## Accomplishments

- Green end-to-end dispatch (run 24623672816) on target commit 973aeff: build + smoke-test + publish + lint-workflow all `conclusion: success` — proves the `needs: [build, smoke-test]` publish gate wired by Plan 04-01 works on a real run, not just in the YAML.
- Live GitHub Release at https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v0.3.20-cu126-win with the Windows CUDA wheel as an asset. Byte-identical upload verified: asset SHA-256 `47649ad0f027fe7364a1a64d4f4e5d7bc2e49d75f471e6c894e3fb7c8adbeed8` matches the publish-job `wheel_sha256` logged value.
- DOC-05 satisfied on rendered github.com page: 12-char llama.cpp SHA `227ed28e128e` appears (a) in the narrative first line ("built 2026-04-19 against llama.cpp @ `227ed28e128e`"), (b) in the `llama.cpp SHA` row of the forensics table, and (c) in the wheel asset filename — all three in sync, proving `steps.meta.outputs.llama_sha_12` substitution took effect (not the 7-char `needs.build.outputs.llama_sha`).
- Visual checkpoint surfaced a content gap: the release body did not disclose that the wheel bundles the CUDA runtime. Remediated in-flight by patching both the live release (via `gh release edit`) AND the workflow template (so the next dispatch doesn't regress), committed as d52a043 + 1d1876b.
- Dispatch-verification plan convention established and respected: the three runtime artifacts (run-meta.json, release.json, publish-log.txt) are declared under `runtime_artifacts` in plan frontmatter, not under `files_modified` — keeps frontmatter-schema linting honest while still tracking the forensics.

## Task Commits

Per-task and pre-flight commits (chronological on branch `main`, fast-forwarded to `windows-build`):

1. **Pre-flight: GSD auto-chain flag sync** - `e110c8d` (chore)
2. **Pre-flight: ignore actionlint + local .claude agent config** - `e58d938` (chore)
3. **SC2012 remediation (dispatch-blocking fix on first attempt):** - `973aeff` (fix) — `ls | grep .whl` flagged by shellcheck SC2012 in the lint-workflow job. Replaced with a nullglob array so we match wheels by glob, not by parsing ls output. This is the commit the green release targets (`targetCommitish: 973aeff4441c38ac7e9c78e6db15518623b34dc1`).
4. **Task 1: Capture dispatch forensics for green publish run** - `9bb1da6` (docs) — saved run-meta.json + release.json + publish-log.txt under `.planning/phases/04-publish-consumer-ux/`.
5. **Task 2 content fix: Disclose CUDA runtime bundling in release body** - `d52a043` (docs) — workflow template patch (so future dispatches render the disclosure) + `gh release edit` on the live v0.3.20-cu126-win body (so consumers see it now).
6. **Task 2 forensics refresh after CUDA disclosure edit** - `1d1876b` (docs) — re-captured `04-02-release.json` so the archived body matches what github.com now shows.

**Plan metadata (this commit):** `docs(04-02): complete dispatch verification plan`

## Files Created/Modified

**Runtime forensics (new):**
- `.planning/phases/04-publish-consumer-ux/04-02-run-meta.json` — conclusion/status/per-job timings for dispatch run 24623672816.
- `.planning/phases/04-publish-consumer-ux/04-02-release.json` — rendered release body + asset metadata (tagName, name, body markdown, assets[].digest + .size, targetCommitish).
- `.planning/phases/04-publish-consumer-ux/04-02-publish-log.txt` — step-by-step publish-job log (used to verify byte-identical wheel upload).
- `.planning/phases/04-publish-consumer-ux/04-02-SUMMARY.md` — this summary.

**Source edits:**
- `.github/workflows/build-wheels-cuda-windows.yaml` — SC2012 wheel-detection fix (973aeff); CUDA-runtime-bundling disclosure baked into release-body template (d52a043).
- `.gitignore` — ignore local `actionlint` binary + `.claude/` agent config (e58d938).

## Decisions Made

- **Dispatch against `windows-build` branch** (not `main`) per user direction. After the run went green, both branches were fast-forwarded on origin to the same HEAD (`1d1876b`) so the fork is consistent regardless of which branch consumers clone.
- **SC2012 remediation via Option A (nullglob array)** instead of more-invasive alternatives (explicit glob pattern with find, shellcheck disable comment, etc.). One-line change, zero schema impact, re-dispatch-friendly.
- **Two-path CUDA disclosure patch**: edit the live release body for consumers who arrive now + update the workflow template for next dispatch. Alternatives considered: (a) edit only the live release (rejected — next dispatch would silently regress the disclosure), (b) edit only the template + wait for next dispatch (rejected — leaves current release body misleading for however long until the next dispatch).

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed SC2012 wheel-detection failure in lint-workflow job**
- **Found during:** First dispatch attempt (pre-Task 1)
- **Issue:** lint-workflow job used `ls dist/*.whl | grep ...` which shellcheck SC2012 flags — and the Phase 1 actionlint integration treats SC2012 as a hard fail. First dispatch went red on lint-workflow before build even finished.
- **Fix:** Replaced with `shopt -s nullglob; wheels=(dist/*.whl)` array pattern. Matches by glob (robust to filenames with spaces, unlike `ls | grep`).
- **Files modified:** `.github/workflows/build-wheels-cuda-windows.yaml`
- **Verification:** Re-dispatched from commit `973aeff`; lint-workflow job went green on the replacement run (24623672816).
- **Committed in:** `973aeff` (fix)

**2. [Rule 2 - Missing Critical] Added CUDA-runtime-bundling disclosure to release body**
- **Found during:** Task 2 (Checkpoint: Visual inspection of rendered Release body)
- **Issue:** User spotted during visual inspection: the release body did not disclose that the wheel bundles the CUDA 12.6 runtime. Consumers scanning the release page would reasonably conclude they needed to install the CUDA Toolkit 12.6.3 locally (matching the `CUDA version: 12.6.3` forensics row), which is both unnecessary and a significant install-barrier ask. This is a critical content gap for DOC-01/DOC-05 accuracy, not a cosmetic nit.
- **Fix:** Two-path remediation — (a) `gh release edit v0.3.20-cu126-win --notes-file <patched-body>` on the live release so consumers see the disclosure immediately, (b) workflow template updated so the next dispatch renders the disclosure natively (no need to re-patch live releases going forward). Disclosure placed at position 2 of the body (right after the narrative first line, before the Install section) so it's seen on scan.
- **Files modified:** `.github/workflows/build-wheels-cuda-windows.yaml` (template); live release body via `gh release edit` (not a file edit).
- **Verification:** Re-ran `gh release view v0.3.20-cu126-win --json body` → confirmed the bundled-runtime paragraph appears. Re-captured as `04-02-release.json` in commit `1d1876b`.
- **Committed in:** `d52a043` (content fix) + `1d1876b` (forensics refresh).

---

**Total deviations:** 2 auto-fixed (1 blocking SC2012 fix, 1 missing-critical content disclosure).
**Impact on plan:** Neither required scope change. SC2012 was a pre-existing lint the workflow had inherited from 04-01 and was surfaced only on first dispatch; the CUDA-bundling disclosure was a genuine documentation gap that strengthens DOC-01/DOC-05 evidence. Both committed individually and folded into the verified release.

## Issues Encountered

- First dispatch attempt went red on lint-workflow (SC2012) before Task 1 could collect any forensics. Handled as a pre-flight remediation (commit 973aeff), after which the second dispatch went fully green. Recorded here rather than as a planned issue because the original plan assumed lint-workflow was already green from 04-01 — the SC2012 rule was introduced by a later actionlint version-bump on this workstation.
- No other issues. Dispatch wall-time (~2h 1m build) was within the Phase 2 envelope; smoke-test + publish + lint-workflow ran fast as predicted in the plan's Expected duration notes.

## Authentication Gates

None triggered during this plan. `gh auth status` was green from prior Phase 4 work; no credential setup was needed for `gh workflow run`, `gh run view`, `gh release view`, `gh release edit`, or `gh release download`.

## User Setup Required

None — no external service configuration required for this plan. The release artifact itself is a user-setup downstream of this repo (consumers download the .whl manually and `pip install path/to/wheel.whl`), but that's DOC-01 documentation shipped in Plan 04-01, not a user-setup for the repo maintainer.

## Self-Check

Verifying all SUMMARY claims before proceeding to state updates.

**Files exist on disk:**
- `.planning/phases/04-publish-consumer-ux/04-02-run-meta.json` — FOUND
- `.planning/phases/04-publish-consumer-ux/04-02-release.json` — FOUND
- `.planning/phases/04-publish-consumer-ux/04-02-publish-log.txt` — FOUND
- `.planning/phases/04-publish-consumer-ux/04-02-SUMMARY.md` — FOUND (this file)

**Task commits exist on disk (git log):**
- `e110c8d` chore(gsd): sync _auto_chain_active flag — FOUND
- `e58d938` chore(gitignore): ignore actionlint + .claude — FOUND
- `973aeff` fix(04-02): SC2012 nullglob array — FOUND (also = release targetCommitish)
- `9bb1da6` docs(04-02): capture dispatch forensics — FOUND
- `d52a043` docs(04-02): CUDA disclosure — FOUND
- `1d1876b` docs(04-02): refresh release forensics — FOUND

**Live artifacts match claims:**
- Release URL https://github.com/ThongvanAlexis/llama-cpp-python/releases/tag/v0.3.20-cu126-win — present (per `04-02-release.json.url`)
- Asset SHA-256 `47649ad0f027fe7364a1a64d4f4e5d7bc2e49d75f471e6c894e3fb7c8adbeed8` — present (per `04-02-release.json.assets[0].digest`)
- 12-char llama.cpp SHA `227ed28e128e` — present in body narrative line, forensics table row, AND asset filename (three matching occurrences)
- Dispatch run URL https://github.com/ThongvanAlexis/llama-cpp-python/actions/runs/24623672816 — referenced in release body forensics table

## Self-Check: PASSED

## Next Phase Readiness

- Phase 4 is now ready for `/gsd:verify-work 04` to close out the v1 milestone formally.
- Both Phase 4 plans (04-01, 04-02) green on disk + on a real dispatch. v1 success criteria from ROADMAP.md Phase 4 "Success Criteria" all satisfied.
- Outstanding Phase 3 requirements (ST-01..ST-08) are marked Complete in REQUIREMENTS.md from the Phase 3 execution (SUMMARY was logged outside this plan's scope); `/gsd:verify-work` will confirm end-to-end.
- No blockers carried forward.

**Next step pointer:** Run `/gsd:verify-work 04` to close the phase and mark the v1 milestone complete.

---
*Phase: 04-publish-consumer-ux*
*Completed: 2026-04-19*
