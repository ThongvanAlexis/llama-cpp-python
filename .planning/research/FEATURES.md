# Feature Research

**Domain:** Windows CUDA wheel-building CI + pip-compatible index publishing (fork of a PyPI package)
**Researched:** 2026-04-15
**Confidence:** HIGH (patterns directly observed from reference implementations: abetlen/llama-cpp-python's index, PyTorch's download index, jllllll's archived index, and PEP 503)

---

## Scope Note

This FEATURES.md covers a **fork CI** — not a general-purpose public index. The downstream consumer is one team (APHP project) that needs `pip install llama-cpp-python` to Just Work against a published Windows CUDA wheel. This framing matters: many "features" that make sense for PyPI, Anaconda, or a commercial vendor's wheelhouse are overkill here and appear in the anti-features section.

Three reference implementations are continuously compared:

| Ref | Structure | Lessons |
|-----|-----------|---------|
| **abetlen's original index** (`abetlen.github.io/llama-cpp-python/whl/cuXXX/`) | gh-pages HTML index + wheels stored as GitHub **Release assets** (`.../releases/download/<version>-cu124/<wheel>`). One flat `index.html` per CUDA variant listing all versions descending, plus embedded per-version subheadings. | This is the UX our fork must match — same URL shape, same install command. Release-asset hosting keeps gh-pages branch small (only HTML, no binaries); critical because 200+ MB wheels on a git branch are painful. |
| **PyTorch's index** (`download.pytorch.org/whl/cuXXX/`) | PEP 503-compliant simple index; wheels hosted on AWS CloudFront; variant is encoded in path (`/whl/cu124/`, `/whl/cpu/`, `/whl/rocm6.1/`) AND in local version suffix (`torch==2.x.y+cu124`). | Variant-in-path is the industry convention. Local version suffix (`+cu124`) is a differentiator we can skip — abetlen doesn't use it and our consumer doesn't need it. |
| **jllllll's archived fork index** (`jllllll.github.io/llama-cpp-python-cuBLAS-wheels/`) | Organized by CPU instruction set (AVX/AVX2/AVX512/basic) rather than CUDA version. Frozen at 0.2.26 since Jan 2024. | **Did right**: filled a maintainer gap the same way we're about to. **Did wrong**: over-segmented the matrix (4 AVX × multiple CUDA × multiple Python = exploded, unmaintainable). Went cold because one human couldn't sustain it. Our lesson: **keep the matrix small, manual, and boring**. |

---

## Feature Landscape

### Table Stakes (Users Expect These)

Two audiences with two expectation sets: **pip-install users** (consume the index) and **CI operators** (us, fixing the workflow). Both sets are non-negotiable.

#### A. Pip-install table stakes (index side)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **PEP 503-compliant HTML index** | `pip install --extra-index-url ...` must be able to discover and download the wheel. Non-compliant HTML breaks pip silently or with confusing errors. | LOW | Single `index.html` at `whl/cu124/llama-cpp-python/` with one `<a>` per wheel file. Normalized package name (`llama-cpp-python`, not `llama_cpp_python`) in URL path. Trailing slashes on directory URLs. |
| **Variant-keyed URL path** (`/whl/cu124/`) | Users select CUDA variant by path, matching both abetlen and PyTorch UX. Install command must be `--extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu124`. | LOW | Static directory layout on gh-pages. Same as abetlen's scheme. |
| **Correct wheel filename tags** | pip filters wheels by `<python_tag>-<abi_tag>-<platform_tag>`. `cp311-cp311-win_amd64` for CPython 3.11 Windows x64. Wrong tag = wheel is silently skipped. | LOW | scikit-build-core emits this automatically; we just must not rename. |
| **Stable wheel asset URLs** | Install reproducibility: the same `pip install X==version` must download the same bytes months later. | LOW | GitHub Release assets are stable (URLs don't change once tagged). Do **not** regenerate releases in place. |
| **Multiple versions coexist** | Users may pin to an older version. Index must list every historical build, not just the latest. | LOW | Append-only pattern: new build = new entry in `index.html`, never delete old entries unless security reason. |
| **Hash integrity fragments** (`#sha256=...`) | PEP 503 recommends hashes in the href; pip uses them for integrity checking when requirements are hash-pinned. | LOW | Compute SHA256 in CI, append `#sha256=<hex>` to href. Trivial step but often skipped. |
| **HTTPS-served index** | `pip` rejects HTTP indexes by default. | LOW | GitHub Pages is HTTPS by default. Free. |
| **README with install instructions** | Users arriving at the repo need to know the `--extra-index-url` command. Without it, the index is technically working but functionally invisible. | LOW | 5-line snippet in README.md showing `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu124`. |

#### B. CI-side table stakes (builder side)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Runtime smoke test gates publish** | PROJECT.md's entire raison d'être: upstream's failure mode was wheels that install but segfault on `Llama(...)`. A publish without smoke test is a publish of garbage. | MEDIUM | Install wheel in a fresh venv on the Windows runner, load `tiny-llama.gguf` (CPU-only OK), generate 2 tokens, assert no crash. Must run **after** wheel build and **before** publish step. |
| **Reproducible-enough builds** | Same inputs (commit, CUDA version, VS version, Python version) → equivalent wheel. Full bit-for-bit reproducibility is aspirational; "same behavior, verified via smoke test" is the bar. | MEDIUM | Pin VS version via `vs-version` range, pin CUDA version via dispatch input, pin Python via setup-python version input, pin runner image (`windows-2022` or `windows-2025`, not `windows-latest`). |
| **Cache restoration on build failure** | Called out explicitly in user spec §3: most dev iterations will be failing builds; don't re-pay the ~3 GB CUDA download every attempt. Default `actions/cache@v4` only saves on success. | MEDIUM | Use `hendrikmuhs/ccache-action` (saves cache in `post` step regardless of job outcome) for sccache. For `actions/cache@v4` on CUDA zip and mamba pkgs, split restore/save: use `actions/cache/restore` at start, `actions/cache/save` with `if: always()` at end. |
| **Build logs retention** | When a build fails at step 47 of 52, we need the full log to diagnose VS/nvcc mismatches. GitHub Actions retains logs 90 days by default — good enough. | LOW | Default behavior; no action needed. Consider `actions/upload-artifact@v4` for the wheel itself so the artifact can be inspected even if publish is skipped. |
| **Wheel artifact upload (pre-publish)** | Even if smoke test fails and we skip publish, we want the built wheel available for local inspection/debugging. | LOW | `actions/upload-artifact@v4` right after build, before smoke test. Retention 7 days is fine. |
| **Version identification in wheel** | Wheel filename must encode the source version (e.g., `llama_cpp_python-0.3.20-cp311...`). Without this, two builds from two upstream versions are indistinguishable. | LOW | Already automatic via scikit-build-core reading `pyproject.toml`. We just don't break it. |
| **Manual trigger (workflow_dispatch)** | PROJECT.md explicit constraint for v1. Avoids runner-minute burn while VS/nvcc fight is being stabilized. | LOW | Already present in upstream workflow. Keep as-is for v1. |
| **Dispatch inputs for Python + CUDA version** | Allows rebuilding for different combos without editing YAML. | LOW | PROJECT.md §Active explicitly requires. Defaults: Python 3.11, CUDA 12.4.1. |
| **Ban `-allow-unsupported-compiler`** | Historical runtime-segfault generator (issue #1543). Banning it forces us to actually fix the VS pin, which is the real problem. | LOW | Just don't add the flag to `CMAKE_ARGS`. Call it out explicitly in a comment in the workflow so future maintainers don't "helpfully" reintroduce it. |

---

### Differentiators (Competitive Advantage)

These are nice-to-haves. For a fork CI, "competitive advantage" is loose — mostly this means "things that make the pipeline more trustworthy or more convenient, but aren't required for v1".

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Sigstore / PEP 740 attestations** | Cryptographic proof that the wheel was built by our GitHub Actions workflow from the claimed commit. PyPI now supports this natively for Trusted Publishing. For a fork, mostly useful against typosquat concerns. | MEDIUM | `pypa/gh-action-pypi-publish` generates attestations automatically when Trusted Publishing is set up. For gh-pages hosting (not PyPI), would need manual `sigstore-python` step. Probably v2 — our consumer is one internal team, not the general public. |
| **SBOM generation (CycloneDX or SPDX)** | Supply-chain transparency: list of all C/CUDA dependencies vendored in the wheel. PEP 770 (Seth Larson, early 2026) standardizes shipping SBOMs inside Python packages. Relevant for security-conscious downstream consumers. | MEDIUM | `cyclonedx-bom` CLI generates from the build environment. Attach as release asset. Not required for APHP consumer today; becomes relevant if the fork ever gets external consumers. |
| **delvewheel DLL vendoring** | `auditwheel` equivalent for Windows — bundles CUDA DLLs into the wheel so it's self-contained. | HIGH | llama-cpp-python historically does NOT delvewheel its CUDA DLLs (the wheel expects the user to have CUDA runtime installed or compatible driver). Matching upstream behavior is probably correct — delvewheel-ing CUDA DLLs would bloat the wheel by 500+ MB and create its own compat headaches. Mark as investigate-v2-only. |
| **Multi-Python matrix in one workflow** | Rebuild 3.10/3.11/3.12/3.13 in one dispatch. | LOW | Deferred in PROJECT.md (v1 = manual dispatch, one combo per run). Matrix expansion is straightforward once the single-combo path is stable. Likely v1.5. |
| **Multi-CUDA matrix in one workflow** | Same idea, for `cu121`/`cu124`/`cu126`/etc. | LOW | Same deferral logic. v1.5 once single-combo is green. Note the VS compatibility wall at CUDA 12.5+ (commit `4f17ae5`) — matrix expansion must be cautious here. |
| **Automatic trigger on upstream tag** | When `abetlen/llama-cpp-python` tags `v0.3.21`, our fork's CI detects and builds. | MEDIUM | Deferred to v2 in PROJECT.md. Requires either a cron job polling upstream tags or a webhook. Both are fragile; not worth it until the manual path is rock-solid. |
| **Build badge in README** | "CI: passing" badge at top of README signals trust. | LOW | One-line markdown include of GitHub Actions SVG badge. Trivial addition. |
| **Changelog of built versions** | A `CHANGELOG.md` on gh-pages that records which upstream version each wheel corresponds to and which runner image / CUDA combo was used. | LOW | Append-only markdown file, updated by CI. Useful debug trail when "why does 0.3.19 work but 0.3.20 segfault?" comes up. |
| **PEP 658 metadata exposure** (`data-dist-info-metadata`) | Modern pip can fetch just the METADATA file without downloading the 200 MB wheel for resolver inspection. | MEDIUM | Requires extracting `.dist-info/METADATA` from each wheel and adding `data-dist-info-metadata="sha256=..."` to the `<a>` tag. Nice supply-chain hygiene; probably v2. |
| **`data-requires-python` attribute** | Lets pip skip the wheel early if the user's Python doesn't satisfy `requires-python`. | LOW | Read from wheel METADATA during index generation, add to `<a>` tag. Low effort, small win for UX. Candidate for v1.1. |

---

### Anti-Features (Commonly Requested, Often Problematic)

Explicit "do not build this" list. Each of these seemed reasonable at some point in planning but is actively wrong for a fork CI serving one downstream team.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Elaborate web UI for the index** (search, filters, charts) | "Our index should look nicer than abetlen's plain HTML." | Pip doesn't consume UI — it parses PEP 503 HTML. Any UI is extra surface area that breaks pip compatibility, adds JS/build tooling, and requires maintenance. abetlen's index is ugly and works perfectly. | Plain PEP 503 HTML. If a human wants to browse, GitHub's "Releases" page is already a UI. |
| **Auto-PR to upstream** | "Our fix should flow back to abetlen/llama-cpp-python." | PROJECT.md §Out of Scope explicitly rules this out. Upstream is in maintainer pause (issue #2136 Quansight funding offer ignored). Automating PRs to a non-responsive upstream creates zombie PRs that degrade trust in our fork. | None. Just do our own thing in our fork. |
| **Multi-registry mirroring** (PyPI + Anaconda + GitHub Packages + custom S3) | "Distribute everywhere for redundancy." | 4× the publish complexity, 4× the auth setup, 4× the failure surface. Consumer (APHP) uses pip; one index is enough. PyPI publishing for a fork is also socially awkward (name squatting on `llama-cpp-python` would break users; publishing as `llama-cpp-python-cuda-win` creates a naming fork that won't match upstream code). | Single gh-pages index. If redundancy is ever needed, mirror to GitHub Release assets, which we already use. |
| **GPG signing of wheels** (`data-gpg-sig`) | "Signed artifacts are more trustworthy." | PEP 503 GPG signing is essentially dead; the community moved to sigstore/PEP 740. Adding GPG = operational burden (key management) for zero consumer benefit — pip doesn't verify GPG automatically. | If signing matters later, use sigstore (listed under differentiators). |
| **Full Python × CUDA cartesian product by default** | "Build for all users from day one." | 4 Python × 4 CUDA = 16 jobs × 30–45 min each = 8–12 hours of runner minutes per dispatch on a workflow that is *currently broken*. jllllll's index died from exactly this (AVX × CUDA × Python explosion with one human maintainer). | Manual dispatch, one combo per run, widen matrix only after the single-combo path is stably green for a month. |
| **Upstream version auto-pinning / auto-release** | "Always publish latest upstream." | The whole project exists because upstream is unstable on Windows CUDA. Auto-publishing latest upstream would propagate upstream breakage directly to our consumer. | Manual dispatch with explicit version input. Human judgment in the loop. |
| **Automated regression test suite** (full pytest suite on GPU) | "Smoke test is weak; we should run the full test suite." | The runner has no GPU. Full suite requires real hardware (GitHub doesn't offer GPU runners on free tier). Smoke test (CPU-only tiny-llama inference) already catches the historical failure mode (segfault on model load). | Smoke test is the bar. If real GPU testing is needed later, it happens on a developer machine, not in CI. |
| **Wheel stripping / size optimization** | "200 MB wheels are huge; we should strip debug symbols." | Symbol stripping for CUDA/MSVC is fiddly; a botched strip breaks the wheel in subtle ways (crashes only in specific codepaths). Upstream doesn't strip. GitHub Release assets handle 2 GB easily. | Don't strip. Size is not a problem for our consumer. |
| **Signed commits / tag verification on upstream** | "Verify upstream integrity before building." | Upstream doesn't sign. Enforcing signing would mean never building anything. | Accept upstream-as-is. Our smoke test is the trust boundary. |
| **`--allow-unsupported-compiler` as "escape hatch"** | "Useful when VS rotates out from under us." | Historically documented failure mode: compiles, then segfaults at runtime. This is literally why abetlen disabled Windows CUDA in commit `98fda8c`. | Fail the build loudly. Fix the VS pin. Covered in PROJECT.md §Key Decisions. |
| **Cross-compilation to other platforms** | "While we're at it, also build Linux/macOS." | Upstream already publishes these. Duplicating is pointless and doubles our maintenance burden. | Stay Windows-only (PROJECT.md §Out of Scope). |
| **PyPI publishing** (not just gh-pages index) | "PyPI is the 'real' package registry." | Name collision: package is `llama-cpp-python` on PyPI, owned by abetlen. Publishing under the same name = impossible. Publishing under a forked name (`llama-cpp-python-win-cuda`) = breaks existing user `import llama_cpp` code. | gh-pages index is exactly the right scope. |

---

## Feature Dependencies

```
PEP 503 HTML index
    └──requires──> GitHub Pages enabled on fork
    └──requires──> gh-pages branch exists
    └──requires──> Wheel asset URLs (stable hosts)

Wheel asset URLs
    └──requires──> GitHub Release (tagged, assets uploaded)
         └──requires──> CI produces a wheel
              └──requires──> Smoke test passes
                   └──requires──> Build succeeds
                        └──requires──> VS version pinned compatibly with CUDA nvcc
                        └──requires──> CUDA toolkit installed on runner
                        └──requires──> Python setup-python with pinned version
                        └──requires──> llama.cpp submodule committed at known rev

Hash integrity fragments (#sha256=...)
    └──requires──> Hash computed during CI (before index generation)
    └──enhances──> PEP 503 HTML index

data-requires-python attribute
    └──requires──> Wheel METADATA extraction at index-gen time
    └──enhances──> PEP 503 HTML index

PEP 658 data-dist-info-metadata
    └──requires──> Wheel METADATA extraction + SHA256
    └──enhances──> PEP 503 HTML index

Cache restoration on failure
    └──requires──> Split restore/save cache actions (or ccache-action)
    └──enhances──> Build succeeds (speeds iteration, doesn't enable anything new)

Sigstore attestations
    └──requires──> Trusted Publishing config OR manual sigstore-python step
    └──enhances──> Wheel asset URLs (provenance, not install UX)

SBOM generation
    └──requires──> Build succeeds
    └──enhances──> Wheel asset URLs (attached as release asset)

Multi-Python matrix
    └──requires──> Single-combo path stable for a month (trust gate)
    └──enhances──> Build succeeds (parallel combos)

Auto-trigger on upstream tag
    └──requires──> Manual dispatch path stable (trust gate)
    └──conflicts──> Smoke-test-gates-publish (if upstream breaks, auto-trigger propagates breakage)
```

### Dependency Notes

- **PEP 503 HTML index requires GitHub Release asset hosting**: observed directly in abetlen's index — wheels live at `releases/download/<tag>/<wheel>`, gh-pages only holds the HTML. This keeps the gh-pages branch small (KB, not GB). Mimicking this pattern is strongly recommended over hosting wheels directly on gh-pages. Note the caveat from the `pip` issue research: GitHub generates on-the-fly redirects for release assets, which can interact poorly with pip's HTTP cache — but this is a minor annoyance, not a blocker.
- **Smoke test gates publish** is the critical dependency: if smoke test fails, we must NOT create the GitHub Release, must NOT update the gh-pages index, must NOT notify the consumer. This is enforced by workflow step ordering: build → artifact upload → smoke test → [fail-stop-here-if-red] → create release → update index.
- **Cache-on-failure enhances nothing functionally** but dramatically improves dev UX during the VS/nvcc fighting phase. First failing builds hit fresh cache; subsequent failing builds hit warm cache and iterate in 5 minutes instead of 30.
- **Auto-trigger conflicts with smoke-test-gates-publish** in spirit: if an auto-triggered build fails smoke test, nothing publishes (which is correct), but nobody is watching either. Manual dispatch keeps a human in the loop, which matches PROJECT.md's v1 philosophy.

---

## MVP Definition

### Launch With (v1) — minimum viable fork CI

The bar is: **one dispatch produces one working wheel consumable via `pip install ... --extra-index-url ...` by the APHP consumer.**

- [ ] **New workflow file** `.github/workflows/build-wheels-cuda-windows.yaml` with `workflow_dispatch` trigger — PROJECT.md §Active
- [ ] **Dispatch inputs**: `python_version` (default 3.11), `cuda_version` (default 12.4.1) — PROJECT.md §Active
- [ ] **VS 2022 path + `vs-version` range pinned** to a range compatible with nvcc 12.4 — PROJECT.md §Active, user spec §5 and §8
- [ ] **Three caches wired up, saving on failure**: sccache (ccache-action), CUDA installer zip (split restore/save), mamba pkgs (split restore/save) — PROJECT.md §Active, user spec §3
- [ ] **Wheel built successfully** via existing scikit-build-core + CMake path
- [ ] **Wheel artifact uploaded** (`actions/upload-artifact@v4`) for post-mortem even if later steps fail
- [ ] **Smoke test step**: install wheel, load `llm_for_ci_test/tiny-llama.gguf`, generate 2 tokens, assert no crash — PROJECT.md §Active, user spec §9
- [ ] **Smoke test is a publish gate**: if it fails, skip release + index update — PROJECT.md §Key Decisions
- [ ] **`tiny-llama.gguf` committed** at `llm_for_ci_test/tiny-llama.gguf` (27 KB, already exists per user spec) — PROJECT.md §Active
- [ ] **GitHub Release created** per successful build, with wheel attached as asset, tag format `v<version>-cu<CUDAmajorminor>-win` or similar
- [ ] **gh-pages branch maintained** with PEP 503-compliant `whl/cu124/llama-cpp-python/index.html` listing all wheels with hash fragments
- [ ] **SHA256 hash fragments** on each href — PEP 503 compliance
- [ ] **Ban on `-allow-unsupported-compiler`** documented in workflow YAML comment — PROJECT.md §Key Decisions
- [ ] **README blurb** with the install command (`pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu124`) — PROJECT.md §Active

### Add After Validation (v1.x)

Trigger: single-combo path is stable for ~1 month of use by APHP without regression.

- [ ] **`data-requires-python` attribute** on index entries — trivial, improves pip's resolver behavior
- [ ] **Build badge** in README — trust signal
- [ ] **Changelog of built versions** on gh-pages or in repo — debug trail
- [ ] **Multi-Python matrix** (3.10, 3.11, 3.12) in one dispatch — only once single-combo is green
- [ ] **Multi-CUDA matrix** (12.4, 12.6) with careful VS pin per CUDA version — constrained by nvcc/VS compatibility wall

### Future Consideration (v2+)

- [ ] **Sigstore / PEP 740 attestations** — only if fork gets external consumers beyond APHP
- [ ] **SBOM generation** (CycloneDX) — only if security review asks for it
- [ ] **PEP 658 `data-dist-info-metadata`** — marginal pip resolver speedup
- [ ] **Auto-trigger on upstream tag** — only after manual path is boringly reliable
- [ ] **delvewheel investigation** — only if Windows users report missing CUDA DLL errors in the wild
- [ ] **Python 3.13 / free-threaded** — track upstream's own progress here (#2136)

---

## Feature Prioritization Matrix

Scoped to features that are actually candidates (anti-features excluded by definition).

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Wheel built successfully (VS/nvcc pin) | HIGH | HIGH | P1 |
| Runtime smoke test gating publish | HIGH | MEDIUM | P1 |
| PEP 503 HTML index on gh-pages | HIGH | LOW | P1 |
| GitHub Release asset hosting of wheels | HIGH | LOW | P1 |
| Three-tier cache w/ save-on-failure | MEDIUM (dev UX) | MEDIUM | P1 |
| Dispatch inputs for Python + CUDA version | MEDIUM | LOW | P1 |
| SHA256 hash fragments in index | MEDIUM (integrity) | LOW | P1 |
| README install blurb | HIGH (discoverability) | LOW | P1 |
| Ban `-allow-unsupported-compiler` | HIGH (prevents segfault-wheels) | LOW | P1 |
| Wheel artifact upload (pre-publish) | MEDIUM (debug) | LOW | P1 |
| `data-requires-python` attribute | LOW | LOW | P2 |
| Build badge in README | LOW | LOW | P2 |
| Changelog of builds | LOW | LOW | P2 |
| Multi-Python matrix | MEDIUM | LOW (once P1 stable) | P2 |
| Multi-CUDA matrix | MEDIUM | MEDIUM (VS wall) | P2 |
| Sigstore attestations | LOW (for APHP), MEDIUM (if public) | MEDIUM | P3 |
| SBOM generation | LOW (for APHP) | MEDIUM | P3 |
| PEP 658 dist-info-metadata | LOW | MEDIUM | P3 |
| Auto-trigger on upstream tag | MEDIUM | MEDIUM | P3 |
| delvewheel DLL vendoring | UNKNOWN | HIGH | P3 |

**Priority key:**
- **P1** — Must have for launch. If any P1 is missing, the project fails its core-value test (published wheel that actually works at runtime, installable via pip).
- **P2** — Should have shortly after launch. Low cost, incremental value.
- **P3** — Future consideration. Only pursued if downstream consumer needs or ecosystem demands evolve.

---

## Competitor Feature Analysis

| Feature | abetlen original (pre-July-2025) | PyTorch (`download.pytorch.org`) | jllllll (archived) | Our approach |
|---------|----------------------------------|----------------------------------|--------------------|--------------|
| Index format | PEP 503 HTML, flat per-CUDA dir | PEP 503 HTML, per-CUDA dir + CloudFront | PEP 503 HTML, per-AVX dir | **Match abetlen exactly** (per-CUDA dir, gh-pages HTML, release-asset wheels) |
| Wheel hosting | GitHub Release assets | AWS CloudFront | GitHub Release assets | **GitHub Release assets** (like abetlen) |
| Variant encoding | URL path (`/whl/cu124/`) | URL path + local version (`+cu124`) | URL path (`/AVX2/`) | **URL path only** (match abetlen; skip local version suffix) |
| Matrix breadth | 4 Python × 4 CUDA × 2 OS | All Python × all CUDA × all platforms | 4 AVX × N CUDA × N Python | **1 × 1 × 1 at launch**, widen carefully |
| Signing / attestations | None | None historically; sigstore on PyPI recently | None | **None at v1**, revisit at v2 |
| SBOM | None | None public | None | **None** |
| Trigger | Tag push + manual | Internal CI | Manual | **Manual only (v1)**; consider tag-triggered at v2 |
| Smoke test before publish | **None** (this is the core failure mode we're fixing) | Unknown (internal) | Unknown | **Mandatory publish gate** — our key differentiator vs. upstream |
| Cache strategy | Single VS-integration cache | N/A (internal infra) | Unknown | **Three-tier cache** (sccache + CUDA zip + mamba pkgs) with save-on-failure |
| Install command shape | `pip install llama-cpp-python --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cu124` | `pip install torch --index-url https://download.pytorch.org/whl/cu124` | `pip install llama-cpp-python --extra-index-url https://jllllll.github.io/.../AVX2/cu121` | **Exactly mirror abetlen's shape** (swap `abetlen` for our user) |
| Project longevity model | One maintainer, went dormant | Org-funded, sustained | One maintainer, went cold | **Internal-team fork**, explicit narrow scope, sustainability via small matrix |

**Critical UX match point**: `pip install llama-cpp-python --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu124` — identical shape to abetlen's original. Any user who previously used abetlen's index should transition by changing one subdomain. This is the single most important UX anchor.

---

## Sources

Reference implementations (direct observation):
- [abetlen's pip index](https://abetlen.github.io/llama-cpp-python/whl/cu124/llama-cpp-python/) — confirmed PEP 503 HTML + GitHub Release asset URL pattern
- [PyTorch wheel index](https://download.pytorch.org/whl/cu124/) — variant-path convention
- [jllllll archived index](https://jllllll.github.io/llama-cpp-python-cuBLAS-wheels/) — cautionary example of over-broad matrix

Specifications:
- [PEP 503 – Simple Repository API](https://peps.python.org/pep-0503/) — HTML requirements, name normalization, hash fragments
- [PEP 658 – Serve Distribution Metadata in the Simple Repository API](https://peps.python.org/pep-0658/) — `data-dist-info-metadata` attribute
- [PEP 691 – JSON-based Simple API for Python Package Indexes](https://peps.python.org/pep-0691/) — JSON alternative (not used in v1)
- [PEP 700 – Additional Fields for the Simple API](https://peps.python.org/pep-0700/)
- [PEP 740 – Attestation objects on the Simple Index](https://blog.sigstore.dev/pypi-attestations-ga/) — sigstore attestations (GA)

Tools and patterns:
- [actions/deploy-pages](https://github.com/actions/deploy-pages) — GitHub Pages deploy action
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) — alternative gh-pages deploy pattern
- [hendrikmuhs/ccache-action](https://github.com/hendrikmuhs/ccache-action) — sccache cache with save-on-failure behavior
- [adang1345/delvewheel](https://github.com/adang1345/delvewheel) — Windows DLL vendoring (v2+ candidate)
- [CycloneDX Python](https://github.com/CycloneDX/cyclonedx-python) — SBOM generation
- [pypi/pypi-attestations](https://github.com/pypi/pypi-attestations) — PEP 740 attestation tooling

Project context:
- `C:\claude_checkouts\llama-cpp-python\.planning\PROJECT.md` — scope, constraints, key decisions
- `C:\claude_checkouts\llama-cpp-python\.planning\user_specs\001_WINDOWS_CUDA_WHEELS_CI_PLAN.md` — origin plan, including §8 on why upstream disabled Windows CUDA and §9 smoke-test proposal

---
*Feature research for: Windows CUDA wheel-building CI + pip-compatible index publishing*
*Researched: 2026-04-15*
