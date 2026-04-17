# Phase 3: Smoke Test (Publish Gate) - Research

**Researched:** 2026-04-17
**Domain:** GitHub Actions CI smoke testing, sparse checkout, CUDA binary inspection, Windows crash dialog suppression
**Confidence:** HIGH

## Summary

Phase 3 adds a `smoke-test` job to the existing `build-wheels-cuda-windows.yaml` workflow. The job runs on a fresh `windows-2022` runner, downloads the wheel artifact from the build job, installs it into a clean venv, and proves the wheel works by loading a 27 KB GGUF fixture and generating 2 tokens. A `cuobjdump --list-elf` check verifies the expected CUDA architectures are embedded in `ggml-cuda.dll`, compensating for the CPU-only runner having no GPU.

The technical domains are well-understood: `actions/checkout@v4` supports sparse checkout with no-cone mode and negation patterns (to exclude `llama_cpp/` source), `actions/download-artifact@v4` handles inter-job artifact transfer, cuobjdump from the mamba `cuda-toolkit` package inspects ELF sections in the DLL, and Windows Error Reporting can be suppressed via registry keys to prevent crash dialogs from hanging the runner. The inference test runs in a subprocess so segfaults produce a catchable non-zero exit code rather than killing the step process.

**Primary recommendation:** Implement as a single plan -- the smoke-test job, fixture commit, sparse checkout, inference test, cuobjdump check, and forensics are tightly coupled and should be added atomically. The `needs:` gate assertion is deferred to Phase 4 per CONTEXT.md.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **cuobjdump sourcing**: Install full `cuda-toolkit` via mamba on the smoke-test runner (same `conda-incubator/setup-miniconda@v3.1.0` + `mamba install cuda-toolkit` pattern as the build job). cuobjdump is guaranteed present with the full toolkit. Inspect `ggml-cuda.dll` in site-packages AFTER `pip install`.
- **Sparse checkout scope**: Full checkout MINUS `llama_cpp/` directory -- use sparse-checkout to exclude only the source package directory. Must include `llm_for_ci_test/`, `.github/workflows/`, and everything else. Explicit assertion: `Test-Path llama_cpp/__init__.py` must FAIL. Explicit assertion: `Test-Path llm_for_ci_test/tiny-llama.gguf` must PASS. After `import llama_cpp`, assert `llama_cpp.__file__` contains `site-packages` and does NOT contain the repo checkout path.
- **Gate assertion placement**: The `needs: [build, smoke-test]` grep-assert belongs in the `lint-workflow` job. **Deferred to Phase 4**: the assertion only makes sense when the publish job exists.
- **Failure diagnostics**: Full forensics on BOTH green and red paths (matching Phase 1's `$GITHUB_STEP_SUMMARY` pattern). Green summary: table with wheel filename, installed version, import path, cuobjdump arch list, venv Python path, inference exit code, output tokens. Red summary: venv layout, Python path, DLL search path, import resolution trace, ggml-cuda.dll location, cuobjdump output, exception traceback. Capture model's 2 output tokens in the log. Run inference test in a **subprocess** (`python -c '...'`). **Disable Windows Error Reporting crash dialog** via registry keys. **Cross-check installed version**: verify `llama_cpp.__version__` contains the expected `+cu126.ll<sha>` pattern.

### Claude's Discretion
- Exact sparse-checkout configuration syntax (cone mode vs no-cone, actions/checkout sparse-checkout input vs manual git sparse-checkout)
- Step ordering within the smoke-test job (setup mamba first or download artifact first)
- Exact WER registry key paths and values
- venv creation mechanism (`python -m venv` vs virtualenv)
- Exact Python subprocess invocation for the inference test
- Per-step `timeout-minutes` values
- Forensics summary table layout and column order
- Remediation hint wording on assertion failures
- Whether to cache mamba pkgs on the smoke-test runner (probably yes -- same `if: always()` save pattern)

### Deferred Ideas (OUT OF SCOPE)
- **`needs: [build, smoke-test]` grep-assert in lint-workflow** -- deferred to Phase 4 when the publish job is created.
- **Multi-platform smoke test** -- Linux/macOS smoke tests (v2 MX-* scope).
- **GPU-accelerated inference test** -- would require a GPU runner (v2 scope).
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| ST-01 | `llm_for_ci_test/tiny-llama.gguf` (27 KB) committed into repo | File exists locally (untracked, 27488 bytes). Must `git add` + commit. |
| ST-02 | Separate `smoke-test` job on fresh `windows-2022` runner | Standard GH Actions job with `runs-on: windows-2022` and `needs: [build]` |
| ST-03 | Sparse checkout omits `llama_cpp/` source | `actions/checkout@v4` with `sparse-checkout-cone-mode: false` and negation pattern `!llama_cpp/` |
| ST-04 | Download wheel artifact from build job | `actions/download-artifact@v4` with `name: cuda-wheel` (contract from Phase 2) |
| ST-05 | Clean venv + pip install downloaded wheel | `python -m venv` + `pip install *.whl` in fresh venv |
| ST-06 | Inference test: load GGUF, generate 2 tokens, no crash | Subprocess `python -c '...'` with WER suppressed; exit code assertion |
| ST-07 | `cuobjdump --list-elf` verifies CUDA architectures in ggml-cuda.dll | cuobjdump from mamba cuda-toolkit; assert sm_80, sm_86, sm_89, sm_90 in output |
| ST-08 | Publish job has `needs: [build, smoke-test]` | **Deferred to Phase 4** per CONTEXT.md. Phase 3 adds smoke-test job; Phase 4 adds publish + gate. |
</phase_requirements>

## Standard Stack

### Core
| Library/Tool | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `actions/checkout@v4` | v4 | Sparse checkout to exclude `llama_cpp/` source | Built-in sparse-checkout support with cone and no-cone modes |
| `actions/download-artifact@v4` | v4 | Download wheel from build job | Standard inter-job artifact transfer; must match upload-artifact@v4 |
| `conda-incubator/setup-miniconda@v3.1.0` | v3.1.0 | Install mamba + cuda-toolkit (for cuobjdump) | Same pattern as build job; locked by CONTEXT.md |
| `python -m venv` | stdlib | Create isolated venv for wheel installation | Ships with Python; no extra dependency |

### Supporting
| Tool | Purpose | When to Use |
|------|---------|-------------|
| `cuobjdump --list-elf` | Verify CUDA architectures in ggml-cuda.dll | After pip install, on `site-packages/llama_cpp/lib/ggml-cuda.dll` |
| Windows Registry (WER) | Suppress crash dialog | Before inference subprocess to prevent runner hang |
| `python -c '...'` subprocess | Run inference test in isolation | Segfaults produce non-zero exit code instead of killing the step |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `python -m venv` | `virtualenv` | venv is stdlib, no extra install; virtualenv faster but unnecessary for CI |
| no-cone sparse checkout | Manual `git sparse-checkout` commands | actions/checkout@v4 handles it natively; manual commands add complexity |
| Full cuda-toolkit install | `cuda-cuobjdump` package only | CONTEXT.md locks full cuda-toolkit; cuobjdump-only would be ~10x smaller but diverges from decision |

## Architecture Patterns

### Smoke-Test Job Structure
```yaml
smoke-test:
  needs: [build]
  runs-on: windows-2022
  timeout-minutes: 30     # No compilation; just install + test
  defaults:
    run:
      shell: pwsh
  steps:
    # 1. Sparse checkout (exclude llama_cpp/ source)
    # 2. Download wheel artifact
    # 3. Setup Python
    # 4. Setup mamba + install cuda-toolkit (for cuobjdump)
    # 5. Disable WER crash dialog
    # 6. Assert sparse checkout exclusion
    # 7. Create venv + install wheel
    # 8. Assert import resolves to site-packages
    # 9. Assert version string
    # 10. cuobjdump CUDA arch check
    # 11. Inference test (subprocess)
    # 12. Green forensics summary (if: success())
    # 13. Red diagnostics (if: failure())
    # 14. Save mamba cache (if: always())
```

### Pattern 1: Sparse Checkout with Negation (No-Cone Mode)
**What:** Use `actions/checkout@v4` with `sparse-checkout-cone-mode: false` and gitignore-style negation patterns to exclude `llama_cpp/` while keeping everything else.
**When to use:** When you need to exclude a specific directory rather than include specific ones.
**Example:**
```yaml
# Source: actions/checkout README + git-sparse-checkout docs
- uses: actions/checkout@v4
  with:
    sparse-checkout: |
      /*
      !llama_cpp/
    sparse-checkout-cone-mode: false
    fetch-depth: 1
```
**Key insight:** In no-cone mode, patterns follow gitignore syntax. `/*` includes everything at root level; `!llama_cpp/` negates (excludes) that directory. This is the inverse of `.gitignore` semantics -- here `!` means "do not include" rather than "do include despite prior exclusion."

**CRITICAL:** Do NOT use `submodules: recursive` on the smoke-test checkout. The build job already compiled the submodule into the wheel. The smoke-test runner has no use for the vendor source and it would waste checkout time. Also, the sparse checkout must NOT exclude `llm_for_ci_test/` (the GGUF fixture directory).

### Pattern 2: Subprocess Isolation for Crash Safety
**What:** Run the inference test as `python -c '...'` subprocess rather than inline Python, so a segfault/access violation produces a non-zero exit code that pwsh can catch cleanly.
**When to use:** Any test that might trigger a native crash (segfault, access violation) on Windows.
**Example:**
```powershell
# Run inference in subprocess -- segfault returns non-zero instead of killing step
$proc = Start-Process -FilePath python -ArgumentList @(
    '-c',
    'from llama_cpp import Llama; llm = Llama(model_path="llm_for_ci_test/tiny-llama.gguf", n_ctx=64); output = llm("hi", max_tokens=2); print(output["choices"][0]["text"])'
) -NoNewWindow -Wait -PassThru
if ($proc.ExitCode -ne 0) { throw "Inference test failed with exit code $($proc.ExitCode)" }
```
**Alternative approach (simpler, captures output):**
```powershell
$output = python -c @"
from llama_cpp import Llama
llm = Llama(model_path='llm_for_ci_test/tiny-llama.gguf', n_ctx=64)
output = llm('hi', max_tokens=2)
print(output['choices'][0]['text'])
"@ 2>&1
if ($LASTEXITCODE -ne 0) { throw "Inference test failed (exit $LASTEXITCODE): $output" }
```

### Pattern 3: WER Crash Dialog Suppression
**What:** Set Windows registry keys before running potentially-crashing subprocesses to prevent the "program has stopped working" dialog from hanging the CI runner indefinitely.
**When to use:** Before any subprocess that might segfault on a Windows CI runner.
**Example:**
```powershell
# Source: Microsoft Learn SetErrorMode docs + WER registry docs
# Disable WER crash dialog -- prevents modal dialog from hanging CI runner
$werKey = 'HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting'
New-ItemProperty -Path $werKey -Name 'DontShowUI' -Value 1 -PropertyType DWORD -Force | Out-Null
New-ItemProperty -Path $werKey -Name 'Disabled' -Value 1 -PropertyType DWORD -Force | Out-Null

# Also set ErrorMode to suppress GPF dialog box for this process tree
# SEM_FAILCRITICALERRORS (0x0001) | SEM_NOGPFAULTERRORBOX (0x0002) = 0x0003
$ErrorMode = 0x0003
Add-Type -TypeDefinition @"
using System.Runtime.InteropServices;
public class WinAPI {
    [DllImport("kernel32.dll")]
    public static extern uint SetErrorMode(uint uMode);
}
"@
[WinAPI]::SetErrorMode($ErrorMode) | Out-Null
```

### Anti-Patterns to Avoid
- **Running inference inline (not subprocess):** A segfault in the same process kills the entire GH Actions step with an opaque "process terminated" error. Subprocess isolation gives a clean exit code.
- **Using cone-mode sparse checkout with negation:** Cone mode only accepts directory paths to include, not negation patterns. Must use `sparse-checkout-cone-mode: false`.
- **Checking `llama_cpp/` source into the sparse checkout:** Defeats the purpose of ST-03 -- the import must resolve from `site-packages`, not the repo checkout.
- **Using `submodules: recursive` on smoke-test:** Wastes time cloning the llama.cpp submodule when only the pre-built wheel is needed.
- **Skipping WER suppression:** A segfaulting wheel will hang the runner indefinitely waiting for a dialog click that never comes.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Sparse checkout | Manual `git sparse-checkout init/set` commands | `actions/checkout@v4` `sparse-checkout` input | Action handles init, cone/no-cone mode, cleanup |
| Artifact transfer | Upload to external storage | `actions/download-artifact@v4` | Integrated, same-workflow, name-matched to upload |
| CUDA arch inspection | Parse DLL binary headers manually | `cuobjdump --list-elf` | NVIDIA's official tool, handles fatbin format correctly |
| Crash dialog suppression | Hope the runner doesn't hang | WER registry keys + SetErrorMode | Registry is the only reliable way on Windows CI |

## Common Pitfalls

### Pitfall 1: Sparse Checkout Cone Mode vs No-Cone Mode
**What goes wrong:** Using the default `sparse-checkout-cone-mode: true` with negation patterns silently fails -- cone mode only accepts directory names to include, not gitignore-style patterns.
**Why it happens:** `actions/checkout@v4` defaults to cone mode (`true`), and the documentation doesn't emphasize that negation patterns require no-cone mode.
**How to avoid:** Explicitly set `sparse-checkout-cone-mode: false` when using `!directory/` negation patterns.
**Warning signs:** All files are checked out despite sparse-checkout being set (cone mode ignores unrecognized patterns silently).

### Pitfall 2: cuobjdump Expecting sm_90 from Virtual Architecture
**What goes wrong:** Build uses `90-virtual` which produces PTX (not ELF/cubin), so `cuobjdump --list-elf` will NOT show `sm_90` for the virtual target -- only `90-real` produces an ELF entry.
**Why it happens:** The build configuration is `80-real;86-real;89-real;90-real;90-virtual`. Both `90-real` AND `90-virtual` are present, so `sm_90` WILL appear in `--list-elf` output (from `90-real`). The `90-virtual` adds PTX for forward compatibility but doesn't remove the ELF.
**How to avoid:** Assert exactly these four in `--list-elf` output: `sm_80`, `sm_86`, `sm_89`, `sm_90`. Do NOT assert `compute_90` (that's PTX, shown by `--list-ptx`).
**Warning signs:** If the build ever drops `90-real` and keeps only `90-virtual`, `sm_90` would disappear from `--list-elf`.

### Pitfall 3: DLL Path After Wheel Install
**What goes wrong:** Looking for `ggml-cuda.dll` at the wrong path after pip install.
**Why it happens:** The wheel installs DLLs to `site-packages/llama_cpp/lib/` (per CMakeLists.txt `RUNTIME DESTINATION`), not to `site-packages/llama_cpp/` root or a system directory.
**How to avoid:** After pip install, locate the DLL at: `<venv>/Lib/site-packages/llama_cpp/lib/ggml-cuda.dll`
**Warning signs:** `cuobjdump` reports "file not found" -- check the exact path.

### Pitfall 4: venv Python Path on Windows
**What goes wrong:** Activating venv and then calling `python` still picks up the system/mamba Python instead of the venv Python.
**Why it happens:** Mamba activation modifies PATH aggressively; venv's `Scripts/` directory must come first.
**How to avoid:** After creating the venv, explicitly use the venv's python path: `<venv>/Scripts/python.exe`. Or activate with `<venv>/Scripts/Activate.ps1` and verify `(Get-Command python).Source` starts with the venv path.
**Warning signs:** `pip install` succeeds but `import llama_cpp` finds the wrong package or finds nothing.

### Pitfall 5: Sparse Checkout State Persistence on Self-Hosted Runners
**What goes wrong:** Sparse checkout configuration from one workflow run persists and affects subsequent runs.
**Why it happens:** `actions/checkout` has a known issue (#2249) where sparse-checkout config persists across workflow runs on self-hosted runners.
**How to avoid:** Not a concern for GitHub-hosted `windows-2022` runners (fresh VM each run). Only matters for self-hosted.
**Warning signs:** N/A for this project (using GitHub-hosted runners).

### Pitfall 6: Access Violation Dialog Hangs Runner
**What goes wrong:** A segfaulting wheel pops up the "program has stopped working" dialog on Windows, which blocks the CI runner indefinitely (no one to click OK).
**Why it happens:** Windows Error Reporting (WER) is enabled by default on the runner image.
**How to avoid:** Set `DontShowUI=1` and `Disabled=1` under `HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting` BEFORE running the inference subprocess. Also call `SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX)`.
**Warning signs:** Smoke-test job times out at the inference step with no log output.

## Code Examples

### Sparse Checkout Configuration (Exclude llama_cpp/)
```yaml
# Source: actions/checkout README, git-sparse-checkout docs
- name: 'Checkout (sparse -- exclude llama_cpp/ source)'
  uses: actions/checkout@v4
  timeout-minutes: 5
  with:
    sparse-checkout: |
      /*
      !llama_cpp/
    sparse-checkout-cone-mode: false
    fetch-depth: 1
    # NO submodules: recursive -- smoke-test doesn't need vendor source
```

### Sparse Checkout Assertions
```powershell
# ST-03 proof: source directory excluded, fixture present
if (Test-Path 'llama_cpp/__init__.py') {
    throw "SPARSE CHECKOUT FAILED: llama_cpp/ source is present. Import would resolve to repo, not site-packages."
}
if (-not (Test-Path 'llm_for_ci_test/tiny-llama.gguf')) {
    throw "SPARSE CHECKOUT FAILED: llm_for_ci_test/tiny-llama.gguf not found. Fixture excluded by sparse checkout."
}
Write-Host "OK: sparse checkout excludes llama_cpp/, includes llm_for_ci_test/"
```

### Venv Creation and Wheel Install
```powershell
# Create clean venv (ST-05)
python -m venv .smoke-venv
$venvPython = Join-Path (Resolve-Path '.smoke-venv') 'Scripts\python.exe'

# Install the downloaded wheel
$wheel = (Get-ChildItem *.whl | Select-Object -First 1).FullName
& $venvPython -m pip install $wheel
if ($LASTEXITCODE -ne 0) { throw "pip install failed (exit $LASTEXITCODE)" }
```

### Import Path Assertion
```powershell
# Verify import resolves to site-packages, not repo checkout
$importPath = & $venvPython -c "import llama_cpp; print(llama_cpp.__file__)"
if ($LASTEXITCODE -ne 0) { throw "import llama_cpp failed (exit $LASTEXITCODE)" }
if ($importPath -notmatch 'site-packages') {
    throw "llama_cpp.__file__ does not contain 'site-packages': $importPath"
}
$repoRoot = (Get-Location).Path
if ($importPath.StartsWith($repoRoot)) {
    throw "llama_cpp.__file__ resolves to repo checkout path: $importPath"
}
Write-Host "OK: llama_cpp imported from $importPath"
```

### Version Cross-Check
```powershell
# Verify installed version contains expected +cu126.ll<sha> suffix
$version = & $venvPython -c "import llama_cpp; print(llama_cpp.__version__)"
if ($version -notmatch '\+cu126\.ll[a-f0-9]+') {
    throw "Version '$version' does not match expected +cu126.ll<sha> pattern"
}
Write-Host "OK: installed version $version"
```

### cuobjdump CUDA Architecture Check
```powershell
# ST-07: verify expected CUDA architectures in ggml-cuda.dll
$sitePackages = & $venvPython -c "import site; print(site.getsitepackages()[0])"
$dllPath = Join-Path $sitePackages 'llama_cpp\lib\ggml-cuda.dll'

if (-not (Test-Path $dllPath)) {
    throw "ggml-cuda.dll not found at $dllPath"
}

$elfOutput = cuobjdump --list-elf $dllPath 2>&1 | Out-String
Write-Host "cuobjdump output:`n$elfOutput"

$expectedArchs = @('sm_80', 'sm_86', 'sm_89', 'sm_90')
$missing = @()
foreach ($arch in $expectedArchs) {
    if ($elfOutput -notmatch $arch) {
        $missing += $arch
    }
}
if ($missing.Count -gt 0) {
    throw "Missing CUDA architectures in ggml-cuda.dll: $($missing -join ', '). Expected: $($expectedArchs -join ', ')"
}
Write-Host "OK: all expected CUDA architectures present: $($expectedArchs -join ', ')"
```

### Inference Subprocess Test
```powershell
# ST-06: load model, generate 2 tokens, capture output
# Runs in subprocess so segfault = non-zero exit, not step death
$inferenceScript = @"
import sys, json
from llama_cpp import Llama
llm = Llama(model_path='llm_for_ci_test/tiny-llama.gguf', n_ctx=64)
output = llm('hi', max_tokens=2)
text = output['choices'][0]['text']
print(f'INFERENCE_OUTPUT={text}')
sys.exit(0)
"@

$output = & $venvPython -c $inferenceScript 2>&1 | Out-String
if ($LASTEXITCODE -ne 0) {
    throw "Inference test FAILED (exit $LASTEXITCODE). Output:`n$output"
}
Write-Host "Inference test passed. Output:`n$output"
```

### WER Suppression
```powershell
# Disable Windows Error Reporting crash dialog
# Must run BEFORE any subprocess that might segfault
$werKey = 'HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting'
New-ItemProperty -Path $werKey -Name 'DontShowUI' -Value 1 -PropertyType DWORD -Force | Out-Null
New-ItemProperty -Path $werKey -Name 'Disabled' -Value 1 -PropertyType DWORD -Force | Out-Null
Write-Host "WER DontShowUI=1, Disabled=1"

# Also set process error mode for subprocess inheritance
# SEM_FAILCRITICALERRORS=0x0001, SEM_NOGPFAULTERRORBOX=0x0002
Add-Type -TypeDefinition @"
using System.Runtime.InteropServices;
public class ErrorMode {
    [DllImport("kernel32.dll")]
    public static extern uint SetErrorMode(uint uMode);
}
"@
[ErrorMode]::SetErrorMode(0x0003) | Out-Null
Write-Host "SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX) applied"
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `actions/checkout@v3` sparse-checkout | `actions/checkout@v4` with `sparse-checkout` + `sparse-checkout-cone-mode` inputs | v4 (2023) | Native sparse-checkout support without manual git commands |
| `download-artifact@v3` (single-run) | `download-artifact@v4` (immutable, must match upload@v4) | v4 (2023) | v4 artifacts are isolated, versioned; v3/v4 incompatible |
| Hope crash dialog doesn't appear | Registry WER suppression + SetErrorMode | Always needed on Windows CI | Only reliable approach; GitHub-hosted runners don't suppress WER by default |

## Open Questions

1. **cuobjdump Windows DLL support**
   - What we know: NVIDIA docs say cuobjdump works with "host executable/object/library" on Windows. The DLL is a standard PE file with embedded CUDA fatbin.
   - What's unclear: Whether cuobjdump from mamba's cuda-toolkit handles `.dll` extension gracefully (vs requiring `.exe` or explicit `.cubin`).
   - Recommendation: First dispatch will validate. If cuobjdump fails on `.dll`, try renaming or extracting the fatbin section. LOW risk -- cuobjdump on DLLs is a well-documented pattern in CUDA development.

2. **Mamba cache savings on smoke-test runner**
   - What we know: The build job caches mamba pkgs with `actions/cache@v4` and `if: always()` save. The smoke-test runner installs the same cuda-toolkit package.
   - What's unclear: Whether the cache key should be shared or separate from the build job's cache.
   - Recommendation: Use a separate cache key prefix (`mamba-smoke-*`) to avoid collision with the build job's cache. The build job may install additional packages. Use the same `if: always()` save pattern.

3. **Tiny-llama.gguf behavior with CUDA wheel on CPU-only runner**
   - What we know: The wheel includes CUDA kernels, but the runner has no GPU. llama.cpp should fall back to CPU execution.
   - What's unclear: Whether the CUDA-built llama.cpp silently falls back to CPU or requires explicit `n_gpu_layers=0`.
   - Recommendation: Explicitly pass `n_gpu_layers=0` in the Llama constructor to ensure CPU fallback. The tiny model's 2-token generation will complete in <1s on CPU.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | GitHub Actions workflow dispatch (empirical) |
| Config file | `.github/workflows/build-wheels-cuda-windows.yaml` |
| Quick run command | `gh workflow run build-wheels-cuda-windows.yaml` |
| Full suite command | Same -- single workflow dispatch runs all jobs |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ST-01 | tiny-llama.gguf committed | smoke/manual | `git ls-files llm_for_ci_test/tiny-llama.gguf` | Wave 0: commit file |
| ST-02 | Separate smoke-test job on windows-2022 | integration | `gh workflow run` + check job list | Wave 0: add job to YAML |
| ST-03 | Sparse checkout omits llama_cpp/ | integration | Assert `Test-Path llama_cpp/__init__.py` = false in CI | Wave 0: add assertion step |
| ST-04 | Download wheel artifact | integration | `actions/download-artifact@v4` step succeeds in CI | Wave 0: add step |
| ST-05 | Clean venv + pip install | integration | pip install exit code = 0 in CI | Wave 0: add step |
| ST-06 | Inference test passes | integration | Subprocess exit code = 0 in CI | Wave 0: add inference step |
| ST-07 | cuobjdump shows expected CUDA archs | integration | cuobjdump output contains sm_80,86,89,90 in CI | Wave 0: add cuobjdump step |
| ST-08 | Publish job gated on smoke-test | integration | Phase 4 grep-assert (deferred) | Phase 4 |

### Sampling Rate
- **Per task commit:** Local YAML lint (`actionlint`) + visual inspection of job DAG
- **Per wave merge:** Full dispatch: `gh workflow run build-wheels-cuda-windows.yaml`
- **Phase gate:** Dispatch must show smoke-test job green, all assertions passing

### Wave 0 Gaps
- [x] `llm_for_ci_test/tiny-llama.gguf` -- already exists locally, needs `git add` + commit
- [ ] `smoke-test` job definition in workflow YAML -- must be added
- [ ] All assertion steps -- must be added to the workflow

## Sources

### Primary (HIGH confidence)
- [actions/checkout@v4](https://github.com/actions/checkout) - sparse-checkout and sparse-checkout-cone-mode inputs documented in README
- [git-sparse-checkout documentation](https://git-scm.com/docs/git-sparse-checkout) - no-cone mode negation pattern syntax (`!directory/`)
- [actions/download-artifact@v4](https://github.com/actions/download-artifact) - inter-job artifact download, must match upload@v4
- [CUDA Binary Utilities 12.6](https://docs.nvidia.com/cuda/archive/12.6.0/cuda-binary-utilities/index.html) - cuobjdump --list-elf output format: `ELF file N: kernel.sm_XX.cubin`
- [Microsoft SetErrorMode](https://learn.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-seterrormode) - SEM_NOGPFAULTERRORBOX (0x0002) + SEM_FAILCRITICALERRORS (0x0001)
- [WER DontShowUI registry key](https://www.sysadmins.lv/retired-msft-blogs/alejacma/how-to-disable-the-pop-up-that-windows-shows-when-an-app-crashes.aspx) - `HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\DontShowUI=1`
- Existing workflow file (`build-wheels-cuda-windows.yaml`) - mamba setup pattern, established conventions

### Secondary (MEDIUM confidence)
- [conda-forge/cuda-cuobjdump-feedstock](https://github.com/conda-forge/cuda-cuobjdump-feedstock) - cuobjdump available as standalone package, also included in full cuda-toolkit
- [NinjaOne WER guide](https://www.ninjaone.com/blog/enable-or-disable-windows-error-reporting/) - WER `Disabled=1` registry key

### Tertiary (LOW confidence)
- cuobjdump on Windows `.dll` files -- documented as working on "host executable/object/library" but no explicit `.dll` example found. First dispatch will validate.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all tools are well-documented, established GH Actions patterns
- Architecture: HIGH - job structure follows existing build job patterns; sparse checkout well-documented
- Pitfalls: HIGH - WER suppression, sparse checkout cone/no-cone mode, DLL path are verified from official docs
- cuobjdump on DLL: MEDIUM - tool documented for host binaries, but no explicit Windows DLL example found

**Research date:** 2026-04-17
**Valid until:** 2026-05-17 (stable domain -- GH Actions, CUDA tools, Windows registry)
