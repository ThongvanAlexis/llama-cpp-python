---
phase: 04-publish-consumer-ux
plan: 01
subsystem: infra
tags: [github-actions, softprops-action-gh-release, release-asset, ubuntu-latest, smoke-test-gate, lint-workflow, yaml, readme, scope-reduction]

# Dependency graph
requires:
  - phase: 02-build-cache
    provides: "cuda-wheel artifact (actions/upload-artifact@v4, name: cuda-wheel), wheel filename scheme (+cu126.ll<sha12>) with 12-char llama.cpp SHA embedded"
  - phase: 03-smoke-test
    provides: "smoke-test job that, on green, gates publish via needs: [build, smoke-test]"
  - phase: 01-scaffold-toolchain-pinning
    provides: "probe-msvc step outputs (selected_toolset/selected_full_version) + lint-workflow job scaffold with ban-grep + actionlint"
provides:
  - "publish job in build-wheels-cuda-windows.yaml: ubuntu-latest, needs: [build, smoke-test], permissions: contents: write, softprops/action-gh-release@v2 upload of the byte-identical cuda-wheel artifact to a GitHub Release at tag v<base_version>-cu<maj><min>-win"
  - "12-char llama.cpp SHA in release body (parsed from wheel filename via sed; option b of the SHA-width fix, no assert-submodule edit)"
  - "lint-workflow publish-gate assertion: anchored presence-grep ^    needs[:]\\s*\\[\\s*build\\s*,\\s*smoke[-]test\\s*\\] must match exactly 1 line; removing the gate on the publish job becomes a detectable regression"
  - "build job-level outputs block (llama_sha 7-char, selected_toolset, selected_full_version) surfacing step-level outputs for needs.build.outputs.* references"
  - "README.upstream.md preserving the 824-line upstream README byte-for-byte"
  - "Fork-focused README.md (~110 lines) with Install (Windows CUDA) manual-download flow, how-the-wheel-is-built 4-stage pipeline recap, Attribution, License"
  - "REQUIREMENTS.md + PROJECT.md scope-reduction amendments (2026-04-19): PUB-03..PUB-10 + DOC-03 moved to Out of Scope; DOC-01 rewritten to manual-download wording; Core Value rewritten to pip install path/to/wheel.whl"
affects: [04-02 (dispatch-verification checkpoint plan), future-v2-gh-pages-revisit]

# Tech tracking
tech-stack:
  added:
    - "softprops/action-gh-release@v2 (GitHub Release asset upload, auto-creates release on first tag push)"
    - "actions/download-artifact@v4 (Phase 2 pair; name: cuda-wheel — singular)"
  patterns:
    - "12-char llama.cpp SHA parsed from wheel filename (`+cu<N>.ll<12hex>-` segment) via sed; keeps assert-submodule's 7-char output untouched (option b of the SHA-width fix — least-invasive)"
    - "Anchored presence-grep for structural gates (^    needs[:]\\s*\\[...smoke[-]test\\]), paired with character-class self-match guards on both the pattern and the echo-string content (use 'colon' / 'dash' paraphrase in error messages to stay regex-invisible)"
    - "Job-level permissions (contents: write on publish only) with workflow-level default read — least-privilege per RESEARCH.md Pitfall 2"
    - "Per-step timeout-minutes on every step of the new publish job (2–5 min)"
    - "Release body assembled via `cat <<BODY ... BODY` heredoc with random delimiter (RELEASE_BODY_$(date +%s)_${RANDOM}) written to GITHUB_OUTPUT as a multi-line output value"
    - "cu-tag derived from inputs.cuda_version via `awk -F. '{printf \"cu%s%s\", $1, $2}'` (not hardcoded — per RESEARCH.md Pitfall 8)"

key-files:
  created:
    - "README.upstream.md (renamed from README.md via git mv; 824 lines, byte-identical to previous README.md)"
  modified:
    - ".github/workflows/build-wheels-cuda-windows.yaml (+161 lines net; new publish job, new lint-workflow step, build job outputs block)"
    - "README.md (overwritten with fork-focused content ~110 lines; previous upstream content preserved at README.upstream.md)"
    - ".planning/REQUIREMENTS.md (DOC-01 rewrite; PUB-03..10 + DOC-03 moved to Out of Scope; traceability status updated; coverage summary updated 14 → 5 active in Phase 4)"
    - ".planning/PROJECT.md (Core Value rewritten to manual-download wording; Key Decisions row added for 2026-04-19 scope reduction)"

key-decisions:
  - "2026-04-19 scope reduction: dropped gh-pages PEP 503 pip index (PUB-03..PUB-10 + DOC-03) in favour of Release-only publishing with manual download; applied via REQUIREMENTS.md Out of Scope move + PROJECT.md Key Decisions row"
  - "12-char llama.cpp SHA for Release body forensics is re-derived from the wheel filename's `+cu126.ll<sha12>-` segment in the publish job's metadata step, NOT propagated through a new assert-submodule output (option b — keeps Phase 1 step untouched)"
  - "Lint-workflow publish-gate assertion uses an anchored ^    needs[:]\\s*\\[... pattern so comments, error messages, and body templates mentioning the phrase in prose don't trigger the presence-count (exactly-1 invariant)"
  - "Error messages inside the lint step use 'colon' / 'dash' paraphrase (needs colon [build, smoke-dash-test]) to stay regex-invisible to any future grep-assert pattern the phrase might land inside"
  - "Publish job runs on ubuntu-latest with no actions/checkout step — all inputs come from download-artifact + needs.build.outputs.*; saves ~3s and reduces attack surface"
  - "make_latest: 'true' (string per action.yml) on softprops so the latest green publish wins the 'Latest' badge; overwrite_files defaults to true for clobber-on-re-dispatch semantics"

patterns-established:
  - "Pattern: Scope-reduction amendment flow — requirements move to Out of Scope table (with consolidated rationale row), Traceability status flips to 'Deferred to v2 / Out of Scope', coverage counts updated, PROJECT.md Key Decisions gets a dated row, ids remain defined (not deleted) so a v2 revisit has the requirement text intact."
  - "Pattern: Last-job in a workflow uses `sed -n '/^  job:/,$p'` for content capture (awk range pattern /^  job:/,/^  [a-z][a-z-]*:/ fails on the last job because the start pattern matches the end pattern on the very first line — gotcha for future plan verify blocks)."
  - "Pattern: Multi-line GITHUB_OUTPUT value uses random-delimiter heredoc to prevent collision with literal EOF inside body text (RESEARCH.md Pitfall 3)."

requirements-completed:
  - PUB-01
  - PUB-02
  - DOC-01
  - DOC-02

# Metrics
duration: ~8 min
completed: 2026-04-18
---

# Phase 4 Plan 01: Publish Job + Consumer UX Summary

**Windows CUDA wheel now ships to the Releases page on a green smoke-test via softprops/action-gh-release@v2, with a grep-asserted publish-gate, a fork-focused README, and a 2026-04-19 scope reduction (gh-pages PEP 503 index dropped; PUB-03..10 + DOC-03 moved to Out of Scope).**

## Performance

- **Duration:** ~8 min (469 s)
- **Started:** 2026-04-18T23:03:21Z
- **Completed:** 2026-04-18T23:11:10Z
- **Tasks:** 3
- **Files modified:** 4 (workflow YAML, README.md, REQUIREMENTS.md, PROJECT.md) + 1 file created (README.upstream.md via git mv preserving the 824-line upstream README byte-for-byte)

## Accomplishments

- `.github/workflows/build-wheels-cuda-windows.yaml` grew a 5-step `publish:` job (ubuntu-latest, needs: [build, smoke-test], contents: write) that downloads the Phase 2 `cuda-wheel` artifact byte-for-byte and uploads it to a GitHub Release at tag `v<base_version>-cu126-win` via `softprops/action-gh-release@v2`, with a 10-row forensics body (12-char llama.cpp SHA, base version, CUDA version, MSVC toolset, Python, wheel size + MiB, wheel SHA-256, build date UTC, dispatch run URL, smoke-test green).
- The build job gained a top-level `outputs:` block surfacing `llama_sha` (7-char, diagnostic), `selected_toolset`, and `selected_full_version` so the publish job can reference `needs.build.outputs.selected_toolset` in the forensics table.
- `lint-workflow` gained a third step — anchored presence-grep `^    needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]` — that fails if the publish-gate `needs: [build, smoke-test]` is ever removed from the publish job's 4-space-indent declaration line; comments and prose that mention the phrase do not trigger the count thanks to the leading-whitespace anchor.
- README.upstream.md preserves the 824-line upstream README byte-for-byte via `git mv`; the fresh README.md (~110 lines) is fork-focused with an intro explaining the fork's purpose, an Install (Windows CUDA) section with the `≥ 561.17` NVIDIA driver floor and manual-download + `pip install path\to\wheel.whl` flow, a how-the-wheel-is-built 4-stage pipeline recap, Attribution, and License.
- REQUIREMENTS.md + PROJECT.md now reflect the 2026-04-19 scope reduction: PUB-03..PUB-10 and DOC-03 moved to Out of Scope (with consolidated rationale row and traceability status flipped to Deferred to v2); DOC-01 rewritten to manual-download wording; PROJECT.md Core Value rewritten and a dated Key Decisions row added.
- actionlint passes cleanly on the full 1835-line workflow file.

## Task Commits

Each task was committed atomically:

1. **Task 1: Wave-0 scaffolding — build job outputs + preserve upstream README + README skeleton** — `8441180` (feat)
2. **Task 2: Publish job + lint-workflow publish-gate assertion** — `b94935c` (feat)
3. **Task 3: Fill README.md body + amend REQUIREMENTS.md and PROJECT.md for scope reduction** — `4bb91d9` (docs)

## Files Created/Modified

- `.github/workflows/build-wheels-cuda-windows.yaml` — Added build job-level `outputs:` block (3 outputs), appended `publish:` job (5 steps, 120 lines), added lint-workflow publish-gate assertion step (16 lines). Net +161 lines.
- `README.md` — Overwritten with fork-focused content (5 H2 sections, ~110 lines).
- `README.upstream.md` — Created via `git mv README.md README.upstream.md` (824 lines, byte-identical to pre-phase README.md).
- `.planning/REQUIREMENTS.md` — DOC-01 rewritten; PUB-03..10 + DOC-03 moved to Out of Scope (2 consolidated rows); traceability status updated for 9 IDs; coverage summary updated (Phase 4 active: 14 → 5); publish section header renamed from "GitHub Pages Pip Index" to "GitHub Release asset"; updated footer timestamp.
- `.planning/PROJECT.md` — Core Value paragraph rewritten (no more `--extra-index-url` wording); Key Decisions table gains a dated 2026-04-19 row for the scope reduction; updated footer timestamp.

## Decisions Made

- **12-char SHA re-derivation from wheel filename (option b).** Plan's `<interfaces>` block called out two options for the SHA-width fix — (a) amend the Phase 1 `assert-submodule` step to emit a 12-char output, or (b) parse the 12-char SHA from the Phase 2 wheel filename's `+cu126.ll<sha12>-` segment in the publish job's metadata step. Chose (b): zero edits to the existing 470-line `build:` job body, full fidelity to the CONTEXT.md-locked 12-char forensics requirement, and a bonus sanity-assert on the wheel filename scheme (fails loudly if Phase 2 ever drifts).
- **Anchored presence-grep for the publish-gate assertion.** Initial implementation used the plan's unanchored pattern `needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]`, which matched 6 lines in the file (one comment in the smoke-test header region, two error-string echoes, two publish-job comments, and the actual declaration). Switched to the anchored `^    needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]` — the 4-space indent uniquely identifies the job declaration line. As belt-and-suspenders, paraphrased the error/log strings ("colon"/"dash") so future greps that might look at those strings don't accidentally double-count either.
- **Release body uses `cat <<BODY ... BODY` with random delimiter written into $GITHUB_OUTPUT.** Random delimiter `RELEASE_BODY_$(date +%s)_${RANDOM}` prevents collision with literal EOF text inside forensics-body content — per RESEARCH.md Pitfall 3. The body uses backslash-escaped backticks (`\``) because YAML `run: |` + bash heredoc passes literal backticks through correctly; the `${{ ... }}` GitHub Actions expressions are interpolated by the Actions runner before bash sees the script.
- **Publish job is the last job in the file.** This means `awk '/^  publish:/,/^  [a-z][a-z-]*:/'` range doesn't work for content capture (start pattern matches end pattern on the first line). For this plan's verification I used `sed -n '/^  publish:/,$p'` instead; flagged as a pattern note for future plans.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 — Bug] Publish-gate presence-grep pattern mismatched the anti-self-match invariant**

- **Found during:** Task 2 (while running the post-task static-check invariants — `grep -cE 'needs:\s*\[\s*build\s*,\s*smoke-test\s*\]'` returned 6 matches, not 1)
- **Issue:** The plan's lint step pattern `needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]` uses character-class escapes (`[:]`, `[-]`) to avoid self-matching the step's own source line, which is correct for the regex pattern's own definition. But the workflow file contains the literal phrase `needs: [build, smoke-test]` on 5 OTHER lines that aren't the pattern's source:
  - 1 header comment in the smoke-test job region
  - 2 echo strings inside the lint step itself (the error/remediation messages)
  - 2 comment/prose mentions in the publish job (a header comment + a body-template cell)
  - 1 actual job declaration (the real target)
  With the plan's invariant `MATCHES -ne 1`, the lint step would fail every dispatch (count = 6, required = 1).
- **Fix:** Anchored the pattern to `^    needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]` (4-space indent is unique to the job declaration line — comments start with `  #`, echo strings have leading whitespace from bash indentation not YAML job-body indent, and the body-template cell is inside a heredoc that gets more indent). Also paraphrased the error/log strings to "needs colon [build, smoke-dash-test]" / "restore the publish job declaration line" so future greps that might inadvertently match those strings cannot.
- **Files modified:** `.github/workflows/build-wheels-cuda-windows.yaml` (lint-workflow step body + error messages)
- **Verification:** `grep -cE '^    needs[:]\s*\[\s*build\s*,\s*smoke[-]test\s*\]' .github/workflows/build-wheels-cuda-windows.yaml` returns exactly `1`. Simulated the full lint step logic end-to-end: "OK: publish-gate present (1 match)". actionlint passes.
- **Committed in:** `b94935c` (Task 2 commit)

**2. [Rule 2 — Missing critical correctness] README intro line referenced `--extra-index-url` and failed the negative grep invariant**

- **Found during:** Task 3 (post-task static-check run — `! grep -qE 'extra-index-url' README.md` failed)
- **Issue:** The plan's README prose (Task 3 action 3a) included the intro line "there is no PyPI publish, no `--extra-index-url` index, no support for Linux/macOS", which is informative prose explaining the scope reduction to consumers. But the plan's `<automated>` verify block and the `<success_criteria>` anti-requirement both enforce `! grep -E '\-\-extra-index-url' README.md` (zero matches). The literal flag string in prose triggered the guard.
- **Fix:** Rephrased the intro to "there is no PyPI publish, no PEP 503 pip index on GitHub Pages, no support for Linux/macOS" — same meaning (explains to consumers what's NOT supported), but doesn't use the literal flag string. Preserves the scope-reduction disclosure while satisfying the anti-requirement grep.
- **Files modified:** `README.md` (intro paragraph)
- **Verification:** `! grep -qE 'extra-index-url' README.md` exits 0 (no matches). Full overall verification suite: 22/22 PASS.
- **Committed in:** `4bb91d9` (Task 3 commit)

---

**Total deviations:** 2 auto-fixed (1 bug in structural gate grep, 1 bug in README prose vs anti-requirement guard)

**Impact on plan:** Both fixes necessary for correctness — without them the lint-workflow job would fail every dispatch (making the workflow unusable) and the negative anti-requirement grep would fail. No scope creep; fixes stay within the plan's intent (enforce the publish-gate presence, keep `--extra-index-url` out of the consumer-facing README).

## Issues Encountered

- **`awk` range pattern edge case on the last job in the file.** The plan's verify blocks used `awk '/^  publish:/,/^  [a-z][a-z-]*:/' "$WORKFLOW" | grep -cE '...'` to scope grep matches to the publish job's YAML range. Because `publish:` is now the last job in the file, the start pattern (`/^  publish:/`) matches the end pattern (`/^  [a-z][a-z-]*:/`) on the very first line, so awk emits only the header line. The plan verify blocks would report zero matches for content that's actually present. For this plan's task-verify and overall verification runs I used `sed -n '/^  publish:/,$p'` instead (capture to EOF). This is an executor-side substitution of the verify command, not a fix to the workflow itself — the actual workflow behaviour is correct. Documented in patterns-established so future plans can avoid the same pattern.

## User Setup Required

None — no external service configuration required. The publish job uses the auto-provided `GITHUB_TOKEN` with job-level `permissions: contents: write` for Release asset upload; no PAT needed for same-repo pushes.

## Next Phase Readiness

- Plan 04-01 closes the static-verifiable portion of v1: PUB-01, PUB-02, DOC-01, DOC-02 are structurally in place (actual grep-verified). DOC-05 (Release body contains 12-char llama.cpp SHA) is structurally wired (publish job emits it and substitutes it into the body) but only dispatch-verifiable (plan 04-02).
- Next step: Run `/gsd:execute-phase 04 --plan 02` for the dispatch-verification checkpoint plan (workflow_dispatch a full cycle, confirm a real GitHub Release is created at `v0.3.20-cu126-win` with the wheel asset attached and the forensics body rendered correctly).
- Blockers/concerns: None from 04-01. The one residual risk is that Phase 3 (smoke-test job, ST-01..08) must also be landed before a real dispatch can exercise the `needs: [build, smoke-test]` gate end-to-end — but ROADMAP.md / STATE.md show Phase 3 is the next phase to execute, not Phase 4's problem.

## Self-Check

- `.github/workflows/build-wheels-cuda-windows.yaml`: FOUND (+161 lines, includes publish job + lint step + build outputs)
- `README.md`: FOUND (~110 lines, fork-focused)
- `README.upstream.md`: FOUND (824 lines, byte-identical to pre-phase README.md)
- `.planning/REQUIREMENTS.md`: FOUND (DOC-01 rewritten; PUB-03..10 + DOC-03 in Out of Scope)
- `.planning/PROJECT.md`: FOUND (Core Value rewritten; Key Decisions 2026-04-19 row added)
- Commit `8441180` (Task 1): FOUND (`git log --oneline` contains it)
- Commit `b94935c` (Task 2): FOUND
- Commit `4bb91d9` (Task 3): FOUND
- actionlint: PASS (exit 0, no output)
- Full overall verification: 22/22 PASS

## Self-Check: PASSED

---
*Phase: 04-publish-consumer-ux*
*Plan: 01*
*Completed: 2026-04-18*
