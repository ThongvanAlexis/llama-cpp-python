# v2 Spec: Fully Automatic Toolchain Selection

## Context

In v1, the workflow requires the user to manually pick `cuda_version` (e.g., `12.6.3`).
MSVC is already auto-selected from the runner. But CUDA selection still requires the user
to know the compatibility constraints across three independent systems:

1. **NVIDIA driver** (user's machine) -- sets the CUDA runtime ceiling
2. **llama.cpp** (submodule) -- may not compile with arbitrarily new CUDA
3. **MSVC on the runner** -- already handled by auto-select + compat matrix

The user shouldn't need to know any of this. The workflow should figure it out.

## Goal

The user provides ONE input: their NVIDIA driver version (e.g., `595.79`).
The workflow automatically selects the highest CUDA version that satisfies all constraints,
then auto-selects the best MSVC for that CUDA. The result is the best-performing wheel
the user's system can run, with zero manual version research.

The cuda dll also need to be bundled into the wheel so that the user does not have to install cuda on is machine

Also note that the user is me, we are not building for the world, only for me

## Input model

```yaml
inputs:
  nvidia_driver:
    description: 'NVIDIA driver version (e.g., 595.79). Determines max CUDA runtime.'
    type: string
    required: true
  cuda_version_override:
    description: 'Force a specific CUDA version (bypass auto-select). Leave empty for auto.'
    type: string
    default: ''
  msvc_toolset_override:
    description: 'Force a specific MSVC toolset (bypass auto-select). Leave empty for auto.'
    type: string
    default: ''
```

## Auto-selection algorithm

```
1. Parse nvidia_driver input
2. Look up max CUDA version supported by that driver (from driver-cuda compat table)
3. Detect llama.cpp version from submodule (git describe / CMakeLists.txt)
4. Look up max CUDA version that llama.cpp is known to compile with (from llama-cuda compat table)
5. CUDA ceiling = min(driver_max_cuda, llamacpp_max_cuda)
6. Pick highest CUDA version <= ceiling that is available on the mamba/conda channel
7. Look up _MSC_VER cap for that CUDA version (from cuda-msvc compat table, already in v1)
8. Enumerate installed MSVC toolsets on runner (already in v1)
9. Pick newest MSVC <= cap (already in v1)
10. Build with (selected_cuda, selected_msvc)
```

## Required compatibility tables

Three hardcoded tables are needed, maintained via periodic web research:

### Table 1: NVIDIA driver -> max CUDA version

Maps minimum driver version to CUDA runtime support. NVIDIA publishes this in their
CUDA toolkit release notes. Forward-compatible: newer drivers support all older CUDA.

```
# Source: https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html
# Table 3: CUDA Toolkit and Minimum Required Driver Version
driver_cuda_compat = {
  "525.60":  "12.0",
  "525.85":  "12.1",
  "535.54":  "12.2",
  "545.23":  "12.3",
  "550.54":  "12.4",
  "555.42":  "12.5",
  "560.28":  "12.6",
  # 12.7 was never released (NVIDIA skipped to 12.8)
  "570.86":  "12.8",
  "575.51":  "12.9",
  "580.00":  "13.0",   # verify exact driver version
  "590.00":  "13.1",   # verify exact driver version
  "595.00":  "13.2",   # verify exact driver version
}
# Algorithm: find highest CUDA where user's driver >= minimum driver
# Example: driver 595.79 >= 595.00 -> max CUDA = 13.2
```

**Maintenance:** update when NVIDIA releases a new CUDA toolkit (roughly quarterly).

### Table 2: llama.cpp version -> max CUDA version

Maps llama.cpp releases to the highest CUDA version they are known to compile with.
This table is built empirically -- either from CI results, community reports, or
explicit testing.

```
# Source: empirical testing + llama.cpp release notes + GitHub issues
# Detection: read vendor/llama.cpp CMakeLists.txt for project version,
#            or use `git -C vendor/llama.cpp describe --tags`
llamacpp_cuda_compat = {
  "b5000+": "12.9",    # verify -- may support 13.x
  "b4000":  "12.8",    # verify
  "b3000":  "12.6",    # verify
  "b2000":  "12.4",    # verify
}
# Algorithm: find llama.cpp version, look up max known-good CUDA
# Conservative: if version is between entries, use the lower bound
```

**Maintenance:** update when bumping the llama.cpp submodule. Test compilation with
the next CUDA version up and extend the table if it works.

**Open question:** can this be auto-detected instead of hardcoded? Possibly by
attempting compilation and catching nvcc errors, but that wastes runner minutes.
A hardcoded table from periodic testing is cheaper.

### Table 3: CUDA version -> max _MSC_VER (already exists in v1)

```
# Source: NVIDIA host_config.h in each CUDA toolkit release
# Already implemented in v1 probe-msvc step
cuda_msvc_cap = {
  "12.2": 1939, "12.3": 1939,
  "12.4": 1949, "12.5": 1949, "12.6": 1949,
  "12.8": 1949, "12.9": 1949,
  "13.0": 1959, "13.1": 1959, "13.2": 1959,  # verify caps for 13.x
}
```

## Workflow output / diagnostics

Every dispatch should log the full decision chain:

```
--- Toolchain Auto-Select ---
NVIDIA driver:     595.79
  -> max CUDA:     13.2 (from driver-cuda table)

llama.cpp version: b5200 (from vendor/llama.cpp)
  -> max CUDA:     12.9 (from llama-cuda table)

CUDA ceiling:      12.9 (min of 13.2, 12.9)
CUDA selected:     12.9.1 (highest available on mamba channel)

MSVC cap for CUDA 12.9: _MSC_VER <= 1949
Installed MSVC:    14.29.30133, 14.44.35207
MSVC selected:     14.44 (_MSC_VER=1944)

Building with: CUDA 12.9.1 + MSVC 14.44
---
```

## What changes from v1

| Aspect | v1 | v2 |
|--------|----|----|
| CUDA version | Manual input (pinned) | Auto-selected from driver + llama.cpp constraints |
| MSVC version | Auto-selected (from CUDA cap) | Same (no change) |
| User input | `cuda_version`, `msvc_toolset` | `nvidia_driver` only |
| Overrides | `msvc_toolset` has override mode | Both `cuda_version_override` and `msvc_toolset_override` |
| Compat tables | 1 (cuda-msvc) | 3 (driver-cuda, llama-cuda, cuda-msvc) |
| Maintenance | Low (cuda-msvc table rarely changes) | Medium (llama-cuda table needs updating per submodule bump) |

## Implementation phases

1. **Research:** web-scrape/verify all three compat tables with exact version numbers
2. **Table encoding:** add tables to the workflow (or a separate YAML/JSON config file
   referenced by the workflow, to keep the workflow readable)
3. **Detection step:** new step before probe-msvc that reads driver input + llama.cpp
   version, resolves CUDA version, then feeds it to the existing mamba install + probe steps
4. **Mamba channel probe:** verify the selected CUDA version's package exists on
   `nvidia/label/cuda-X.Y.Z` before attempting install (fast HTTP HEAD check)
5. **Testing:** dispatch with known driver versions and verify correct CUDA selection

## Edge cases

- **Driver too old for any supported CUDA:** fail with "Your driver X.Y supports max CUDA Z.
  Minimum required: 12.2. Update your NVIDIA driver."
- **llama.cpp version not in table:** warn, fall back to the lowest CUDA in the table
  (conservative). Suggest updating the table.
- **No MSVC on runner compatible with selected CUDA:** already handled by v1 probe.
- **Mamba channel doesn't have the selected CUDA patch:** fall back to next-lower
  available patch (e.g., 12.9.1 not available -> try 12.9.0).
- **Override provided:** skip auto-select for that dimension, validate compat, warn if
  the override is suboptimal ("You forced CUDA 12.4 but your driver supports up to 13.2").

## CUDA architectures as dispatch input

In v1, the CUDA SM architecture list is hardcoded to `80-real;86-real;89-real;90-real;90-virtual`
(Ampere+). For v2, this could become a dispatch input so users can trade wheel size for GPU
coverage (e.g., add `75-real` for Turing/RTX 2000, or drop older archs to shrink the wheel
further). The 400MB wheel size limit (BLD-11) constrains the upper bound.

## Non-goals for v2

- Multi-CUDA matrix builds (building cu124 + cu126 + cu128 in one dispatch) -- v3
- Auto-detecting the user's driver version from their machine -- out of scope for CI
- Bundling CUDA DLLs into the wheel (delvewheel) -- separate v2 spec if needed
