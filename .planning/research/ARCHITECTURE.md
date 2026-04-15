# Architecture Research

**Domain:** GitHub Actions CI pipeline — Windows CUDA wheel build + PEP 503 pip index on GitHub Pages
**Researched:** 2026-04-15
**Confidence:** HIGH (GitHub Actions primitives, PEP 503 format), MEDIUM (optimal split of the DAG — several defensible shapes)

Scope reminder: the llama-cpp-python *library* architecture is already covered in `.planning/codebase/ARCHITECTURE.md`. This file is exclusively about the *CI pipeline architecture* — what jobs exist, how artifacts flow, how caches compose, and how the gh-pages index is structured. The downstream consumer is the roadmap, which needs phase-level component boundaries and data-flow contracts.

---

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       Trigger  (workflow_dispatch)                        │
│    inputs: python_version=3.11, cuda_version=12.4.1, ref=<branch|tag>     │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
         ┌─────────────────────────────┴────────────────────────────┐
         │                        JOB 1: build                       │
         │  runs-on: windows-2022                                    │
         │  steps:                                                   │
         │    1. checkout (recursive submodules)                     │
         │    2. setup-python (pyver input)                          │
         │    3. setup-msbuild (pinned VS range for nvcc 12.4)       │
         │    4. restore caches  ◄─────┐                             │
         │       - sccache              │ keys composed from         │
         │       - CUDA installer zip   │ (os, pyver, cuda,          │
         │       - conda pkgs           │  submodule SHA, CMake hash)│
         │       - VS MSBuildExtensions │                             │
         │    5. install CUDA toolkit (mamba)                        │
         │    6. copy MSBuildExtensions into VS 2022                 │
         │    7. derive version from llama_cpp/__init__.py           │
         │    8. python -m build --wheel  → dist/*.whl               │
         │    9. save caches (always, even on failure)               │
         │   10. upload-artifact: wheel (name=wheel-<ver>-<pyver>-cu<cu>)│
         │                                                           │
         │  outputs: wheel_name, version, cuda_tag                   │
         └──────────────────────────────────────┬────────────────────┘
                                                │ needs:
                                                ▼
         ┌──────────────────────────────────────────────────────────┐
         │                    JOB 2: smoke-test                      │
         │  runs-on: windows-2022   (fresh runner, fresh venv)       │
         │  steps:                                                   │
         │    1. checkout (sparse: just llm_for_ci_test/*.gguf)      │
         │    2. setup-python (same pyver from needs)                │
         │    3. download-artifact: wheel                            │
         │    4. pip install <wheel>.whl  (NO repo source on path)   │
         │    5. python -c "from llama_cpp import Llama; \           │
         │                   Llama('tiny-llama.gguf', n_ctx=64); \   │
         │                   llm('hi', max_tokens=2)"                │
         │    6. fail loudly on import error, load error, segfault   │
         │                                                           │
         │  This job is a publish gate. Red here → no gh-pages push. │
         └──────────────────────────────────────┬────────────────────┘
                                                │ needs: (only if green)
                                                ▼
         ┌──────────────────────────────────────────────────────────┐
         │                   JOB 3: publish                          │
         │  runs-on: ubuntu-latest  (cheap, no CUDA needed)          │
         │  steps:                                                   │
         │    1. checkout gh-pages branch                            │
         │    2. download-artifact: wheel                            │
         │    3. place wheel at whl/cu<tag>/llama-cpp-python/<file>  │
         │    4. regenerate whl/cu<tag>/llama-cpp-python/index.html  │
         │       (one <a> per wheel in that folder, PEP 503)         │
         │    5. commit & push gh-pages (peaceiris/actions-gh-pages) │
         └───────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component (job) | Owns | Does NOT own |
|---|---|---|
| `define_matrix` (optional) | Expanding `workflow_dispatch` inputs into a matrix of (pyver × cuda) jobs when more than one combo is requested. | Any build work. For v1 (single combo) this job can be dropped and inputs wired directly into `build`. |
| `build` | Wheel production. CUDA toolkit install, VS integration, cache restore/save, `python -m build --wheel`, emitting `dist/*.whl` as an artifact. | Running the wheel. The build job never does `import llama_cpp` from the freshly built wheel — it would pick up the in-tree source instead of the wheel. |
| `smoke-test` | Validating that the wheel installs into a *clean* environment and a `Llama(...)` call does not segfault. Publishes a pass/fail signal. | Building anything. It consumes an artifact; if there is no artifact, it does not run. |
| `publish` | Materializing the gh-pages directory: placing the wheel under `whl/cu<tag>/llama-cpp-python/` and regenerating `index.html` so it conforms to PEP 503. | Building, testing, or re-validating the wheel. If it was good enough to ship past smoke-test, publish is a pure file-move + index regen. |

The key boundary is **smoke-test ≠ build**. A wheel that compiles is not a wheel that works — that is precisely the upstream failure mode (`-allow-unsupported-compiler` → runtime segfaults per issues #1543/#1551). Splitting the jobs forces a fresh Python environment where the in-tree `llama_cpp/` source is not importable, so `from llama_cpp import Llama` either resolves to the installed wheel or fails. Same-job validation cannot guarantee this — the checkout puts the source directory on `sys.path`.

---

## Recommended Project Structure

```
.github/
└── workflows/
    └── build-wheels-cuda-windows.yaml   # new file; does NOT replace build-wheels-cuda.yaml
                                          # keeps upstream merges conflict-free

llm_for_ci_test/
└── tiny-llama.gguf                       # 27 KB fixture, committed. Consumed by smoke-test.

.planning/
├── codebase/                             # library architecture (already exists)
└── research/
    └── ARCHITECTURE.md                   # this file
```

### Why a single workflow file (not `build.yaml` + `publish.yaml`)?

| Option | Pros | Cons |
|---|---|---|
| **Single file, multi-job** (recommended) | Atomic run: one dispatch → build + test + publish or none; single `needs` DAG is easy to read; artifact passing stays inside the run (no cross-workflow `workflow_run` plumbing); concurrency group scoped to one entity. | Longer YAML. One dispatch cannot "publish the previous run" without a second entry point. |
| Split (`build.yaml` → `publish.yaml` via `workflow_run`) | Re-publish without rebuild; publish logic reusable across triggers. | Cross-workflow artifacts require either `actions/upload-artifact` + `actions/download-artifact@v4` with `run-id` + `github-token` across workflows, or release-asset staging. More moving parts; harder to reason about "did the thing publish?". |

The v1 scope (`workflow_dispatch` only, manual runs) makes the single-file shape strictly better. Split becomes interesting only if a future v2 introduces automatic tag-triggered builds with a separate manual "republish stable artifact" path — not a v1 concern.

### Structure Rationale

- **New file, not editing `build-wheels-cuda.yaml`:** Upstream still owns the multi-OS file. Keeping our Windows-only variant in a separate filename means `git pull upstream main` never generates conflicts in CI config. Confirmed decision in PROJECT.md.
- **Fixture committed in-repo (`llm_for_ci_test/tiny-llama.gguf`, 27 KB):** Avoids runtime download in the smoke-test; removes network as a failure mode for the publish gate. Size is negligible for git.
- **No shell scripts outside the workflow file:** PROJECT.md constraint ("keep debugging surface small"). All PowerShell stays inline in `run:` blocks.

---

## Architectural Patterns

### Pattern 1: Three-Job DAG (build → smoke-test → publish)

**What:** Linear job dependency chain where each job has a single responsibility and the artifact is the handoff contract.

**When to use:** Any pipeline where "it compiled" is a weaker claim than "it works." The whole point of this fork existing is to break the upstream pattern of publishing compiled-but-broken wheels. Splitting smoke-test out is the architectural expression of that goal.

**Trade-offs:**
- **Pro:** Smoke-test on a fresh runner forcibly validates wheel self-sufficiency (no `vendor/llama.cpp/` on disk, no `CMakeLists.txt` to accidentally satisfy an import). This catches the "wheel ships without the native `.dll`" class of bug that a same-job test misses.
- **Pro:** Clean job-level logs. Build log stays build-only; smoke-test log is just `pip install` + `import` + `Llama(...)`. When something fails, you know which layer broke.
- **Pro:** Publish is gated by `needs: smoke-test` — a red smoke test cannot reach gh-pages. The publish job literally cannot run.
- **Con:** ~30-60 s overhead for the runner spin-up and artifact download in the smoke-test job. Acceptable on a build that already takes 5–45 minutes.
- **Con:** A fresh runner costs minutes; in a fast-iterating failure loop (where the *build* keeps failing) you don't pay this cost, because smoke-test only runs when build is green.

**Example (job skeleton):**
```yaml
jobs:
  build:
    runs-on: windows-2022
    outputs:
      wheel_name: ${{ steps.build.outputs.wheel_name }}
      version: ${{ steps.version.outputs.version }}
      cuda_tag: ${{ steps.build.outputs.cuda_tag }}
    steps:
      # ... restore caches, install CUDA, build ...
      - uses: actions/upload-artifact@v4
        with:
          name: wheel-${{ steps.version.outputs.version }}-py${{ inputs.python_version }}-cu${{ steps.build.outputs.cuda_tag }}
          path: dist/*.whl
          retention-days: 7

  smoke-test:
    needs: build
    runs-on: windows-2022
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: llm_for_ci_test
      - uses: actions/setup-python@v5
        with: { python-version: ${{ inputs.python_version }} }
      - uses: actions/download-artifact@v4
        with: { name: wheel-${{ needs.build.outputs.version }}-py${{ inputs.python_version }}-cu${{ needs.build.outputs.cuda_tag }} }
      - run: pip install (Get-ChildItem *.whl).FullName
      - run: |
          python -c "from llama_cpp import Llama; \
                     llm = Llama(model_path='llm_for_ci_test/tiny-llama.gguf', n_ctx=64); \
                     print(llm('hi', max_tokens=2)); \
                     print('SMOKE_OK')"

  publish:
    needs: [build, smoke-test]
    runs-on: ubuntu-latest
    # ... gh-pages publish ...
```

### Pattern 2: Artifact Handoff via `upload-artifact` / `download-artifact` (not job outputs)

**What:** Pass the wheel file itself via the artifact store; pass metadata (version, cuda_tag, wheel_name) via job outputs.

**When to use:** Always, for binary files. Job outputs are capped at ~1 MB total per job and are string-only — useless for a 200 MB wheel. Upload/download is the canonical GitHub-blessed path.

**Trade-offs:**
- **Pro:** Artifact storage is free (within retention limits), scoped to the run, auto-cleaned on retention expiry.
- **Pro:** `actions/upload-artifact@v4` compresses the wheel (already compressed internally, so ~0% gain, but costs nothing).
- **Pro:** `download-artifact@v4` in a subsequent job in the same workflow doesn't need `github-token`/`run-id` — it discovers the artifact by name within the run.
- **Con:** Zipping/unzipping adds ~5–10 s for a 200 MB wheel. Irrelevant next to build time.
- **Con:** File permissions are not preserved across the zip boundary (GitHub docs). Not an issue for `.whl` (zip-format internally, no executable bits needed).

**Anti-pattern:** Base64-encoding the wheel and emitting it as a job output. Hits the 1 MB output ceiling instantly. Don't.

**Metadata does go through outputs:**
```yaml
outputs:
  version: ${{ steps.version.outputs.version }}       # e.g., "0.3.20"
  cuda_tag: ${{ steps.build.outputs.cuda_tag }}       # e.g., "124"
  wheel_name: ${{ steps.find-wheel.outputs.name }}    # e.g., "llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl"
```

### Pattern 3: Version Derivation from `llama_cpp/__init__.py` (single source of truth)

**What:** Parse the version string from the same file that `pyproject.toml` already points at via `scikit_build_core.metadata.regex`, rather than inventing a second source.

**When to use:** Always, in this repo. `pyproject.toml` declares:
```toml
[tool.scikit-build.metadata.version]
provider = "scikit_build_core.metadata.regex"
input = "llama_cpp/__init__.py"
```
`llama_cpp/__init__.py` ends with `__version__ = "0.3.20"`. This is the authoritative version — `python -m build --wheel` will already name the wheel `llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl` based on that line.

**Trade-offs:**
- **Pro:** Zero drift. Whatever is in `__init__.py` is what `scikit-build-core` stamps onto the wheel; derive the same way for tags, artifact names, and gh-pages URLs.
- **Pro:** Works whether the CI is triggered on `main`, a feature branch, or a tag — doesn't require `git tag` to be present.
- **Pro:** One-line extraction:
  ```powershell
  $version = (Select-String -Path llama_cpp/__init__.py -Pattern '__version__\s*=\s*"([^"]+)"').Matches[0].Groups[1].Value
  Write-Output "version=$version" >> $env:GITHUB_OUTPUT
  ```
- **Con:** If someone bumps the tag without bumping `__init__.py`, the wheel carries the old version. This is desirable — wheels must agree with themselves.

**Rejected alternatives:**
- **`git describe` / tag-based:** Couples version derivation to tag hygiene. Fails on untagged branches. Not how the build system derives it, so would drift.
- **`workflow_dispatch` input:** Lets CI lie about the version. Bad.
- **Cross-parse `pyproject.toml`:** Indirect — `pyproject.toml` just says "read `__init__.py`." Skip the middleman.

### Pattern 4: Layered Cache with `restore-keys` Fallback

**What:** Every cache has a precise primary key (full match = fast path) and progressively shorter `restore-keys` (partial match = warm-start path). Save-on-failure is enabled so unfinished-but-partial caches survive red builds.

**When to use:** Always, for this build. Dev-loop rationale from PROJECT.md: "most dev iterations will be failing builds; don't re-pay download cost every attempt."

**Trade-offs:**
- **Pro:** Failed builds still produce a sccache with partial compilation results; next run restarts 70–90 % warm.
- **Pro:** Minor input changes (e.g., touching a `.py` file) miss the primary key but hit a `restore-keys` prefix and restore the last compatible cache.
- **Con:** The cache store has size limits (10 GB per repo on free tier). Multiple variants (pyver × cuda × submodule-SHA) can accumulate. Mitigation: GitHub auto-evicts LRU.
- **Con:** Over-broad `restore-keys` can restore a cache with stale CUDA headers. Mitigation: CUDA version is in every primary key (so restore only hits caches built under the same CUDA).

### Pattern 5: Separate Publish Runner (ubuntu, not windows)

**What:** Jobs 1 and 2 run on `windows-2022`. Job 3 (publish to gh-pages) runs on `ubuntu-latest`.

**When to use:** Anytime the publish step is pure file manipulation + git push. No reason to pay for a Windows runner to `cp` a wheel and regen an HTML file.

**Trade-offs:**
- **Pro:** Linux runner minute cost is ~5× cheaper than Windows.
- **Pro:** Bash for regenerating `index.html` is trivially shorter than PowerShell.
- **Pro:** `peaceiris/actions-gh-pages` is battle-tested on Linux runners.
- **Con:** Three different OSes if we also add a matrix later. Not a real cost.

---

## Data Flow

### Trigger → Wheel → Index

```
[workflow_dispatch: pyver=3.11, cuda_version=12.4.1]
      │
      ▼
┌─── JOB: build ────────────────────────────────────────┐
│ checkout (recursive)                                   │
│   + vendor/llama.cpp @ <submodule-SHA>                 │
│   + CMakeLists.txt                                     │
│   + llama_cpp/__init__.py (version source)             │
│ ↓                                                      │
│ restore caches                                         │
│   - sccache     (key includes submodule SHA)           │
│   - CUDA zip    (key: cuda_version)                    │
│   - conda pkgs  (key: cuda_version + runner.os)        │
│   - VS MSBuildExt (key: cuda_version)                  │
│ ↓                                                      │
│ mamba install cuda-toolkit=<cuda_version>              │
│ ↓                                                      │
│ copy MSBuildExtensions → VS 2022 BuildCustomizations   │
│ ↓                                                      │
│ extract version from llama_cpp/__init__.py             │
│   $version = "0.3.20"                                  │
│ ↓                                                      │
│ python -m build --wheel                                │
│   → dist/llama_cpp_python-0.3.20-cp311-cp311-          │
│                           win_amd64.whl                │
│ ↓                                                      │
│ save caches (always, even on failure)                  │
│ ↓                                                      │
│ upload-artifact: wheel-0.3.20-py3.11-cu124             │
│   path: dist/*.whl                                     │
│   outputs: version=0.3.20, cuda_tag=124,               │
│            wheel_name=llama_cpp_python-…-win_amd64.whl │
└────────────────────────────┬───────────────────────────┘
                             │
                             ▼
┌─── JOB: smoke-test ────────────────────────────────────┐
│ checkout (sparse: llm_for_ci_test only — NO source)    │
│ ↓                                                      │
│ setup-python 3.11                                      │
│ ↓                                                      │
│ download-artifact: wheel-0.3.20-py3.11-cu124           │
│ ↓                                                      │
│ pip install <wheel>.whl                                │
│ ↓                                                      │
│ python -c "from llama_cpp import Llama; \              │
│            Llama('llm_for_ci_test/tiny-llama.gguf',    │
│                  n_ctx=64)                             │
│            ('hi', max_tokens=2)"                       │
│ ↓                                                      │
│ success → green; fail (ImportError, segfault,          │
│            OSError) → red → publish blocked            │
└────────────────────────────┬───────────────────────────┘
                             │ (only on green)
                             ▼
┌─── JOB: publish ───────────────────────────────────────┐
│ checkout gh-pages branch                               │
│ ↓                                                      │
│ download-artifact: wheel-0.3.20-py3.11-cu124           │
│ ↓                                                      │
│ mkdir -p whl/cu124/llama-cpp-python/                   │
│ mv *.whl whl/cu124/llama-cpp-python/                   │
│ ↓                                                      │
│ regenerate whl/cu124/llama-cpp-python/index.html       │
│   (scan dir, emit <a href="file.whl">file.whl</a>)     │
│ ↓                                                      │
│ commit & force-push gh-pages                           │
│   (peaceiris/actions-gh-pages@v4)                      │
└────────────────────────────┬───────────────────────────┘
                             │
                             ▼
   User: pip install llama-cpp-python \
           --extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu124
```

### gh-pages Index Layout (PEP 503)

The directory tree on the `gh-pages` branch:

```
/                                              ← GitHub Pages root
├── index.html                                 ← optional landing (human-readable)
└── whl/
    ├── cu124/
    │   ├── llama-cpp-python/                  ← PEP 503 normalized project name
    │   │   ├── index.html                     ← the pip-facing index
    │   │   ├── llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl
    │   │   ├── llama_cpp_python-0.3.19-cp311-cp311-win_amd64.whl
    │   │   └── llama_cpp_python-0.3.18-cp310-cp310-win_amd64.whl
    │   └── index.html                         ← optional, lists project(s)
    ├── cu125/                                 ← future: different CUDA builds
    └── cu128/
```

**PEP 503 rules relevant to us:**

1. **URL normalization:** Project name "llama-cpp-python" is already the normalized form (lowercase, `.`/`_`/`-` runs collapsed to `-`). `re.sub(r"[-_.]+", "-", name).lower()` on "llama_cpp_python" gives "llama-cpp-python" — which is why the *folder* is `llama-cpp-python/` (hyphens) while the *wheel filename* is `llama_cpp_python-…` (underscores, per PEP 427 wheel naming). Don't confuse the two.
2. **Trailing slash required:** `https://<user>.github.io/llama-cpp-python/whl/cu124/llama-cpp-python/` must resolve to the index (GitHub Pages serves `index.html` for directory URLs by default — matches the spec).
3. **`--extra-index-url` anchor:** User points pip at `https://<user>.github.io/llama-cpp-python/whl/cu124` (without the trailing project segment). pip appends `/llama-cpp-python/` itself based on the package name it's installing.

**`index.html` format** (what the publish job regenerates):

```html
<!DOCTYPE html>
<html>
  <body>
    <a href="llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl"
       data-requires-python="&gt;=3.8">llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl</a><br/>
    <a href="llama_cpp_python-0.3.19-cp311-cp311-win_amd64.whl"
       data-requires-python="&gt;=3.8">llama_cpp_python-0.3.19-cp311-cp311-win_amd64.whl</a><br/>
  </body>
</html>
```

Rules:
- One `<a>` per wheel, `href` is *relative to the index.html* (just the filename is enough since wheels live beside it).
- Link text should match the filename (PEP 503 recommendation — some tools use it as a display-name fallback).
- `data-requires-python` is optional but recommended; set from `pyproject.toml`'s `requires-python = ">=3.8"`. HTML-encode the `>` as `&gt;`.
- Optional hash fragment on href: `#sha256=<hex>` lets pip verify integrity. Worth adding.
- **No `data-yanked`, no `data-dist-info-metadata`** unless we actually produce detached `.metadata` files — out of scope for v1.

**Regeneration logic** (pseudo-bash in the publish job):
```bash
cd whl/cu124/llama-cpp-python
{
  echo '<!DOCTYPE html><html><body>'
  for whl in *.whl; do
    sha=$(sha256sum "$whl" | cut -d' ' -f1)
    echo "<a href=\"$whl#sha256=$sha\" data-requires-python=\"&gt;=3.8\">$whl</a><br/>"
  done
  echo '</body></html>'
} > index.html
```

Idempotent: the wheels on disk are the source of truth; the index is a deterministic projection.

### Key Data Flows

1. **Source → Wheel:** `checkout` + `vendor/llama.cpp` submodule → cached objects (sccache) → `cmake` via scikit-build-core → `dist/*.whl`. The version embedded in the wheel comes from `llama_cpp/__init__.py`.
2. **Wheel → Validation:** `upload-artifact` (build) → `download-artifact` (smoke-test) → `pip install` in a fresh Python → `from llama_cpp import Llama` loads the built-in `.dll` from the wheel. If the `.dll` is missing or dynamically linked to something that isn't on the runner, this is where you find out.
3. **Wheel → Pages:** `download-artifact` (publish) → `mv` to `whl/cu<tag>/llama-cpp-python/` → regenerate `index.html` by scanning the directory → `git commit && git push origin gh-pages`.

---

## Cache Key Composition

This is the most load-bearing part of the architecture — wrong keys mean either cache misses (slow) or poisoned caches (builds succeed with stale artifacts). Each cache has a different "what invalidates me?" answer.

| Cache | Primary Key | Restore-keys (fallbacks) | Invalidation rationale |
|---|---|---|---|
| **sccache** (compiler objects) | `sccache-${{ runner.os }}-py${{ inputs.python_version }}-cu${{ inputs.cuda_version }}-submod${{ hashFiles('vendor/llama.cpp/.git/HEAD', '.gitmodules') }}-cmake${{ hashFiles('CMakeLists.txt', 'pyproject.toml') }}` | `sccache-${{ runner.os }}-py…-cu…-submod…-`<br/>`sccache-${{ runner.os }}-py…-cu…-`<br/>`sccache-${{ runner.os }}-py…-`<br/>`sccache-${{ runner.os }}-` | Objects are the product of source × Python ABI × CUDA ABI × toolchain flags. Any of those changing must miss. Submodule SHA changes = llama.cpp source change = must miss. CMake/pyproject change = compiler flags might change = must miss. |
| **CUDA installer zip** | `cuda-installer-${{ inputs.cuda_version }}-${{ runner.os }}` | `cuda-installer-${{ inputs.cuda_version }}-` | The installer zip at a given CUDA version is immutable (NVIDIA's URL is versioned). No fallback needed beyond OS variants. |
| **conda pkgs dir** | `mamba-cuda-${{ inputs.cuda_version }}-${{ runner.os }}-py${{ inputs.python_version }}` | `mamba-cuda-${{ inputs.cuda_version }}-${{ runner.os }}-`<br/>`mamba-cuda-${{ inputs.cuda_version }}-`<br/>`mamba-cuda-` | conda packages for CUDA X.Y.Z are version-pinned; Python version affects which deps get pulled. |
| **VS MSBuildExtensions** | `cuda-${{ inputs.cuda_version }}-vs-integration` | (none — existing upstream pattern) | The extracted MSBuildExtensions are a pure function of the CUDA version. No hashing needed. |
| **pip cache** | handled by `actions/setup-python@v5` with `cache: 'pip'` | (managed by setup-python) | Keyed on `pyproject.toml`/lockfile by the action. Don't override. |

**Recipe (the essential one — sccache):**

```yaml
- name: sccache
  id: sccache-windows
  uses: hendrikmuhs/ccache-action@v1.2
  with:
    variant: sccache
    key: windows-llama-cpp-cuda-py${{ inputs.python_version }}-cu${{ inputs.cuda_version }}-submod${{ hashFiles('vendor/llama.cpp/.git/HEAD', '.gitmodules') }}-cmake${{ hashFiles('CMakeLists.txt', 'pyproject.toml') }}
    restore-keys: |
      windows-llama-cpp-cuda-py${{ inputs.python_version }}-cu${{ inputs.cuda_version }}-submod${{ hashFiles('vendor/llama.cpp/.git/HEAD', '.gitmodules') }}-
      windows-llama-cpp-cuda-py${{ inputs.python_version }}-cu${{ inputs.cuda_version }}-
      windows-llama-cpp-cuda-py${{ inputs.python_version }}-
      windows-llama-cpp-cuda-
```

**Submodule-SHA nuance:** `actions/checkout@v4` with `submodules: recursive` materializes the submodule at the repo's pinned SHA. `hashFiles('vendor/llama.cpp/.git/HEAD')` returns a stable hash of the `HEAD` pointer file for that pinned commit — changing the submodule pointer in the superproject commit updates this file's contents, so the key invalidates. Gotcha: on a fresh clone `vendor/llama.cpp/.git` may be a *file* pointing into `.git/modules/…` instead of a directory. `hashFiles` handles both (it hashes the file content regardless), but verify on the first run by printing `Get-Content vendor/llama.cpp/.git/HEAD`.

**Save on failure:** Keepassxc uses `actions/cache/save@v5` with `if: "!cancelled() && steps.<id>.outputs.cache-hit != 'true'"`. We want the same — save the cache unless (a) the job was cancelled or (b) we got an exact cache hit to begin with (nothing new to save). `hendrikmuhs/ccache-action` handles this internally. For `actions/cache@v4` on the CUDA zip / conda pkgs, we need the explicit restore+save split pattern from the keepassxc reference.

---

## Scaling Considerations

Unusual for a CI-pipeline research doc, but relevant because "scale" here means "how much does the matrix expand?"

| Scale | Architecture Adjustments |
|---|---|
| **v1: one combo (py 3.11 × cu 12.4.1)** | Single job pair, no `define_matrix` job needed. Dispatch inputs feed `build` directly. ~30–45 min cold, 5–10 min warm. |
| **v1.5: one combo + "matrix when explicitly dispatched"** | Re-introduce the `define_matrix` job (already present in upstream) that reads dispatch inputs into a matrix array. Single combo by default; user can pass a comma-list to expand. |
| **v2: full matrix (3.10/3.11/3.12 × 12.4/12.5/12.8)** | 9 parallel `build` jobs. Each gets its own cache keyspace (py and cu are in the key). smoke-test fanned out to 9 as well. publish must serialize (one gh-pages push) — use `concurrency.group: publish-gh-pages`. |

### Scaling Priorities

1. **First bottleneck: CUDA toolkit install time.** Even with conda cache, ~2–3 min per job. At matrix size 9, that's 27 min of wallclock if serialized — but it parallelizes, so real cost is max(job). Cache saves the per-job variance.
2. **Second bottleneck: gh-pages serialization.** Git can't do concurrent pushes. `concurrency.group: publish-gh-pages` with `cancel-in-progress: false` serializes them — each publish job waits its turn, but the index always reflects the latest push.
3. **Third bottleneck: artifact store quota.** Free tier = 500 MB total artifacts in flight. Nine 250 MB wheels = 2.25 GB. Mitigation: `retention-days: 1` on the build artifacts (publish happens within minutes, not days); wheels on gh-pages are the permanent copy.

Not a v1 concern. Single combo fits comfortably under all limits.

---

## Anti-Patterns

### Anti-Pattern 1: Same-job build + test

**What people do:** Run `pip install dist/*.whl && python -c "from llama_cpp import Llama; …"` as a step in the `build` job, right after the build step.

**Why it's wrong:** The repo checkout is on disk. `sys.path` in the runner's Python includes the current directory. `from llama_cpp import …` resolves to `./llama_cpp/__init__.py` — the *source* — not the installed wheel. You "tested" the source tree with whatever shared library lingered in `./llama_cpp/lib/` from an earlier build step. The wheel's self-containment is never exercised. This is exactly how upstream shipped `-allow-unsupported-compiler` wheels that installed fine and segfaulted at runtime on user machines — the build runner's environment masked the problem.

**Do this instead:** Separate job, sparse checkout that omits `llama_cpp/` from disk, fresh `actions/setup-python@v5`, install wheel from the downloaded artifact. The first line Python runs after `pip install` should be `import sys; print(sys.path)` — verify `llama_cpp` resolves to `site-packages`, not to `.`.

### Anti-Pattern 2: `-allow-unsupported-compiler` in CMake flags

**What people do:** Paper over the nvcc/VS version mismatch with `-DCMAKE_CUDA_FLAGS=--allow-unsupported-compiler` so the build turns green.

**Why it's wrong:** Upstream issue #1543 documented this produces wheels that compile but segfault on first `Llama(...)` call. The flag is currently in the inherited `build-wheels-cuda.yaml` (line 159) and is explicitly banned by PROJECT.md. The smoke-test job is designed to catch cases where this flag would let a bad wheel through — but the right fix is pinning VS version, not tolerating the mismatch.

**Do this instead:** Pin `vs-version: '[17.X,17.Y)'` in `microsoft/setup-msbuild@v2` to a range nvcc 12.4.1 explicitly supports. If the runner image rotates and the pin stops resolving, the build fails loudly — which is what we want. A red build at this boundary is a feature.

### Anti-Pattern 3: Single workflow file with `if:` branches for build-vs-publish

**What people do:** One big job that does everything with `if: startsWith(github.ref, 'refs/tags/')` gates around publish steps (upstream does this — line 173 of `build-wheels-cuda.yaml`).

**Why it's wrong:** No separation between "compiled" and "worked." No clean re-run semantics — "republish without rebuild" is impossible. Publish logic lives inside a job pinned to a Windows runner for no reason. Mixing concerns makes every failure mode harder to diagnose.

**Do this instead:** Three jobs, linear `needs` chain, each job doing one thing. Upstream's pattern is a historical accident, not a recommendation.

### Anti-Pattern 4: Wheels as GitHub Release assets *instead of* gh-pages index

**What people do:** Publish via `softprops/action-gh-release@v2` to a Release; tell users to download the `.whl` and `pip install` it by path.

**Why it's wrong:** Misses the whole UX point. Users want `pip install llama-cpp-python --extra-index-url …` to Just Work. Release-asset UX forces them to hunt for the right wheel, download manually, and pip install from a path — exactly the friction the project is trying to eliminate. Also breaks CI/requirements.txt reproducibility for downstream consumers (the APHP project, per PROJECT.md).

**Do this instead:** gh-pages PEP 503 index is the canonical path. It matches abetlen's original UX and the community's expectation for CUDA-variant wheels. Releases are still fine as a secondary archive, but are not the primary distribution mechanism.

### Anti-Pattern 5: Parsing the wheel filename to get the version

**What people do:** `python -m build --wheel`, then `ls dist/*.whl` and regex out `0.3.20` from the filename to use in the artifact name / tag.

**Why it's wrong:** Circular. The version is read from `llama_cpp/__init__.py` by scikit-build-core to produce the filename; parsing the filename to recover the version is indirection for no reason. Also slows debugging — "why does the artifact name say one thing and the wheel another?" becomes possible.

**Do this instead:** Read `llama_cpp/__init__.py` *before* the build step, export to `GITHUB_OUTPUT`, use it for everything downstream (artifact name, gh-pages path, release tag). Pattern 3 above.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---|---|---|
| NVIDIA CUDA installer | `Invoke-RestMethod` from `Jimver/cuda-toolkit` link registry, cached via `actions/cache@v4` keyed on CUDA version | URLs are stable per version (immutable binaries); fallback by downloading directly from NVIDIA if Jimver's registry changes. |
| conda-forge / nvidia channel | `conda-incubator/setup-miniconda@v3.1.0` + `mamba install -c nvidia/label/cuda-X.Y.Z cuda-toolkit=X.Y.Z` | Rate-limited but generally reliable. Cache `~/conda_pkgs_dir` to reduce round-trips. |
| Visual Studio Build Tools | `microsoft/setup-msbuild@v2` with pinned `vs-version` range | **Critical integration point.** The VS image on `windows-2022` rotates; nvcc's version check is strict. Pin the range to what the target CUDA supports. |
| GitHub Pages | `peaceiris/actions-gh-pages@v4` with `publish_branch: gh-pages`, `publish_dir: ./whl-staging` | Handles the git-commit/force-push dance. Alternative: manual `git` commands if we want fine control. |
| GitHub Artifacts | `actions/upload-artifact@v4` + `actions/download-artifact@v4` | v4 is required (v3 is archived as of 2025). Default 90-day retention; we use 7 days. |

### Internal Boundaries

| Boundary | Communication | Notes |
|---|---|---|
| `build` ↔ `smoke-test` | Artifact (`wheel-<ver>-py<pv>-cu<cv>`) + job outputs (version, cuda_tag) | Artifact is the binary; outputs are the metadata used to find/name the artifact and construct the gh-pages path. |
| `smoke-test` ↔ `publish` | Implicit — `needs: [build, smoke-test]` on publish ensures smoke-test green before publish runs | No data flows directly from smoke-test to publish. Publish re-downloads the same artifact from `build`. |
| `publish` ↔ `gh-pages` branch | `git push` via `peaceiris/actions-gh-pages` | Serialized by `concurrency.group: publish-gh-pages` to avoid push races if the matrix ever expands. |
| CI ↔ upstream `build-wheels-cuda.yaml` | None — new file `build-wheels-cuda-windows.yaml` | Explicitly decoupled per PROJECT.md. Upstream merges never touch our file. |

---

## Summary for the Roadmap

**Component boundaries (what each job owns):**
- `build`: CUDA toolkit + VS integration + CMake + scikit-build-core → `dist/*.whl`. Emits artifact. Does not verify.
- `smoke-test`: Fresh Python, pip install the artifact, load the tiny-llama fixture, generate 2 tokens. Gate for publish.
- `publish`: Move the artifact into the gh-pages tree at `whl/cu<tag>/llama-cpp-python/`, regenerate `index.html` per PEP 503, push.

**Data-flow contract:**
- Wheel binary → artifact store (`upload-artifact` / `download-artifact`).
- Version + cuda_tag → job `outputs` (strings, < 1 MB).
- Final index → git commit on `gh-pages` branch.

**Build order (phase dependencies):**
1. Caches + deps (restore sccache, CUDA zip, conda pkgs; mamba install CUDA).
2. Build (scikit-build-core produces wheel).
3. Smoke-test (fresh runner, wheel from artifact, tiny-llama inference).
4. Publish (gh-pages push with regenerated PEP 503 index).

Each step is a hard prerequisite for the next. Caches save-on-failure so (1) is cheap on retry during the expected VS/nvcc-fighting phase.

**Cache-key recipe (must-include inputs):**
- OS (`runner.os`)
- Python version (`inputs.python_version`)
- CUDA version (`inputs.cuda_version`)
- `vendor/llama.cpp` submodule SHA (`hashFiles('vendor/llama.cpp/.git/HEAD', '.gitmodules')`)
- CMake build config hash (`hashFiles('CMakeLists.txt', 'pyproject.toml')`)

Omitting any of these risks stale caches producing wrong wheels. The compiler-objects cache (sccache) needs all five; the CUDA-installer-zip cache only needs CUDA version + OS (the zip is immutable).

**gh-pages index layout:**
```
whl/cu<tag>/llama-cpp-python/index.html
whl/cu<tag>/llama-cpp-python/llama_cpp_python-<ver>-cp<py>-cp<py>-win_amd64.whl
```
`index.html` contains one `<a href="<wheel>.whl#sha256=…" data-requires-python="&gt;=3.8">` per wheel. User installs via `--extra-index-url https://<user>.github.io/llama-cpp-python/whl/cu<tag>` (pip appends the project path itself).

---

## Sources

- [GitHub Docs — Store and share data with workflow artifacts](https://docs.github.com/actions/using-workflows/storing-workflow-data-as-artifacts) — HIGH confidence (official)
- [actions/upload-artifact v4 README](https://github.com/actions/upload-artifact) — HIGH confidence (official)
- [actions/cache README](https://github.com/actions/cache) — HIGH confidence (official)
- [PEP 503 — Simple Repository API](https://peps.python.org/pep-0503/) — HIGH confidence (authoritative spec)
- [Packaging Python User Guide — Simple repository API (PEP 691 update)](https://packaging.python.org/en/latest/specifications/simple-repository-api/) — HIGH confidence (official)
- [pip documentation — `--extra-index-url`, PEP 503 expectations](https://pip.pypa.io/en/stable/cli/pip_index/) — HIGH confidence (official)
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) — HIGH confidence (widely-used community action)
- [girder/create-pip-index-action](https://github.com/girder/create-pip-index-action) — MEDIUM confidence (useful reference for index regeneration, but we'll inline the logic rather than take the dep)
- [hendrikmuhs/ccache-action](https://github.com/hendrikmuhs/ccache-action) — HIGH confidence (used in keepassxc reference)
- Local reference: `C:\claude_checkouts\keepassxc\.github\workflows\build.yml` — HIGH confidence (direct code, in-repo)
- Local reference: `C:\claude_checkouts\llama-cpp-python\.github\workflows\build-wheels-cuda.yaml` — HIGH confidence (direct code, in-repo)
- Local reference: `C:\claude_checkouts\llama-cpp-python\.planning\user_specs\001_WINDOWS_CUDA_WHEELS_CI_PLAN.md` — HIGH confidence (project constraints)
- Upstream issues #1543, #1551, #1838, #1894 (VS/nvcc compatibility, segfault-after-allow-unsupported) — MEDIUM confidence (historical, referenced in user spec)

---
*Architecture research for: GitHub Actions Windows CUDA wheel CI for llama-cpp-python fork*
*Researched: 2026-04-15*
