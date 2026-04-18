# llama-cpp-python (Windows CUDA fork)

This fork adds a GitHub Actions CI workflow that builds a Windows x64 CUDA 12.6
wheel of [`llama-cpp-python`](https://github.com/abetlen/llama-cpp-python) with
CUDA DLLs pre-packaged inside the wheel (no CUDA toolkit install needed on the
consumer side). Upstream paused Windows CUDA wheels in July 2025 (see
abetlen/llama-cpp-python#1543 and #2136) after `-allow-unsupported-compiler`
shipped wheels that segfaulted at runtime; this fork reactivates the Windows
CUDA path for our own use with a smoke-test publish gate that structurally
prevents the same failure mode.

The only deliverable here is the Windows CUDA wheel on the
[Releases page](https://github.com/ThongvanAlexis/llama-cpp-python/releases) —
there is no PyPI publish, no PEP 503 pip index on GitHub Pages, no support
for Linux/macOS (upstream still ships those). For the upstream Python API
documentation, see [`README.upstream.md`](README.upstream.md).

## Install (Windows CUDA)

**Prerequisites.** You need:

- Windows x64
- Python 3.11 (matches the published wheel's `cp311` tag)
- NVIDIA driver **≥ 561.17** (the driver that ships with CUDA 12.6.3). Check
  with:

  ```
  nvidia-smi
  ```

  The first line of output reports your driver version; anything ≥ `561.17`
  is supported. CUDA 12.x minor-version compatibility technically works as
  low as `527.41`, but `561.17` is the safe figure that matches the build
  toolchain.

**Install steps — manual download from the Releases page.**

1. Go to
   [https://github.com/ThongvanAlexis/llama-cpp-python/releases](https://github.com/ThongvanAlexis/llama-cpp-python/releases)
   and pick the latest `v<version>-cu126-win` tag.
2. Download the `.whl` matching your Python version (e.g.
   `llama_cpp_python-0.3.20+cu126.ll<sha>-cp311-cp311-win_amd64.whl`).
3. Install by pointing `pip install` at the local path to the downloaded
   wheel:

   ```
   pip install path\to\llama_cpp_python-0.3.20+cu126.ll<sha>-cp311-cp311-win_amd64.whl
   ```

4. Verify:

   ```
   python -c "from llama_cpp import Llama; print('ok')"
   ```

   No DLL-load error means the CUDA DLLs inside the wheel were loaded
   correctly and you're ready to inference.

## How the wheel is built

The CI pipeline is a 4-stage workflow in
[`.github/workflows/build-wheels-cuda-windows.yaml`](.github/workflows/build-wheels-cuda-windows.yaml):

1. **Preflight** — MSVC toolset auto-selected from the runner's installed
   versions against a CUDA/`_MSC_VER` compatibility matrix; `nvcc` and
   `cl.exe` versions asserted against the dispatched `cuda_version` /
   expected cap. `-allow-unsupported-compiler` is banned (grep-assert)
   because it's the upstream segfault trap (#1543).
2. **Build** — scikit-build-core invokes CMake with Ninja, sccache, and
   `/Z7` debug info; produces a correctly-tagged wheel with the 12-char
   `llama.cpp` submodule SHA embedded in the version string for
   reproducibility. Size-bounded under 400 MiB.
3. **Smoke-test** — a fresh `windows-2022` runner with sparse checkout (no
   `llama_cpp/` source) installs the wheel, asserts `import llama_cpp`
   resolves to `site-packages`, checks CUDA architectures in
   `ggml-cuda.dll` via `cuobjdump`, and loads a 27 KB tiny-llama GGUF.
   Publish is **structurally gated** on this job passing
   (`needs: [build, smoke-test]`), enforced by a lint-workflow grep-assert
   so removal is a detectable regression.
4. **Publish** — on green smoke-test only, the wheel is uploaded as a
   GitHub Release asset at tag `v<base_version>-cu126-win` with a
   forensics body recording the llama.cpp SHA, MSVC toolset, CUDA version,
   wheel size and SHA-256, and the dispatch run URL.

See the workflow file for the authoritative source.

## Attribution

Upstream project:
[github.com/abetlen/llama-cpp-python](https://github.com/abetlen/llama-cpp-python).
The original upstream README (824 lines of Python API documentation,
OpenAI/LangChain integration examples, Docker usage, server modes) is
preserved byte-for-byte at [`README.upstream.md`](README.upstream.md) for
reference.

## License

Same MIT license as upstream — see [`LICENSE.md`](LICENSE.md).
