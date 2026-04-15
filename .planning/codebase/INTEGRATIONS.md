# External Integrations

**Analysis Date:** 2026-04-15

## APIs & External Services

**OpenAI API Compatibility:**
- Service: OpenAI Chat/Completion endpoints
  - What it's used for: Drop-in replacement API for OpenAI models
  - SDK/Client: OpenAI Python SDK (optional, for `create_chat_completion_openai_v1()`)
  - Location: `llama_cpp/llama.py:create_chat_completion_openai_v1()` method
  - Auth: OpenAI API key (when using actual OpenAI endpoints, not needed for local inference)
  - Notes: Returns ChatCompletion/ChatCompletionChunk objects compatible with OpenAI SDK types

**Hugging Face Hub Integration:**
- Service: Hugging Face Model Hub
  - What it's used for: Model discovery and download from HuggingFace repositories
  - SDK/Client: `huggingface-hub` package (optional dependency, not in core `dependencies`)
  - Location: `llama_cpp/llama.py:from_pretrained()` method
  - Auth: `HF_TOKEN` environment variable (for accessing private models)
  - Test usage: `hf_hub_download()` in `tests/test_llama.py` for downloading test models
  - Typical models: `lmstudio-community/Qwen3.5-0.8B-GGUF`, `CompendiumLabs/bge-small-en-v1.5-gguf`

## Native Library Bindings

**llama.cpp (C/C++ Library):**
- Type: Compiled shared library (vendored as Git submodule)
- Location: `vendor/llama.cpp/` (Git submodule)
- Build: CMake-based compilation with `CMakeLists.txt`
- Binding method: ctypes FFI (`llama_cpp._ctypes_extensions.load_shared_library()`)
- Library loading:
  - Search path: `llama_cpp/lib/` directory (after installation)
  - Override: `LLAMA_CPP_LIB_PATH` environment variable
  - Supported platforms:
    - Linux/FreeBSD: `libllama.so`
    - macOS: `libllama.so` or `libllama.dylib`
    - Windows: `llama.dll` or `libllama.dll`
- Function definitions: ctypes decorators in `llama_cpp/llama_cpp.py`
- Symbol management: RTLD_GLOBAL on Windows for dependency resolution

**LLaVA (Vision) Support:**
- Type: Optional compiled extension for multimodal models
- Build option: `LLAVA_BUILD` (default: ON) in `CMakeLists.txt` line 6
- Binding: `llama_cpp/llava_cpp.py` - ctypes bindings for vision features
- Installation: Compiled to `llama_cpp/lib/` alongside llama library

**MTMD (Multi-task Multi-device) Support:**
- Type: Optional multi-device inference support
- Binding: `llama_cpp/mtmd_cpp.py` - ctypes bindings for distributed inference
- Use case: GPU offloading across multiple devices

## Data Storage

**Databases:**
- Type: No persistent database integration
- Models: Loaded from local filesystem or downloaded from Hugging Face Hub

**File Storage:**
- Approach: Local filesystem only
  - Model files: `.gguf` format (from llama.cpp ecosystem)
  - Cache directory: User-configurable, default system temp or `.cache/`
  - DiskCache storage: `~/.cache/llama-cpp-python/` or custom path

**Caching:**
- Service: DiskCache (local file-based)
  - Implementation: `llama_cpp/llama_cache.py`
  - Types: RAM cache (`LlamaRAMCache`) and disk cache (`LlamaDiskCache`)
  - Configuration: Via `llama_cpp/server/settings.py` - `cache` (bool) and `cache_type` (str: "ram"/"disk")
  - Purpose: Store computed KV states to avoid re-evaluation

## Authentication & Identity

**Auth Provider:**
- Type: Custom or external (configurable)
- Server auth:
  - HTTPBearer implementation available in `llama_cpp/server/app.py` line 20
  - Auth token passed in HTTP headers for REST API endpoints
  - Not enforced by default (optional)
  - Configuration: Via Pydantic Settings (environment variables)

## Monitoring & Observability

**Error Tracking:**
- Service: None (built-in)
- Approach: Standard Python logging via `llama_cpp/_logger.py`
- Logger output: Bridged to C library logging (configure llama.cpp verbosity)

**Logs:**
- Approach: Python standard library `logging` module
  - Logger class: `llama_cpp._logger.LlamaLoggerCallback`
  - Sink: ctypes callback to C library for llama.cpp native logs
  - Verbosity control: `verbose=True/False` parameter in `Llama` class

## CI/CD & Deployment

**Hosting:**
- Deployment target: PyPI package distribution
  - Package name: `llama-cpp-python`
  - Repository: https://github.com/abetlen/llama-cpp-python

**CI Pipeline:**
- Service: GitHub Actions
- Test matrix: Linux (ubuntu-latest), Windows (windows-latest), macOS (macos-latest)
- Python versions tested: 3.9, 3.10, 3.11, 3.12
- Build wheels: Separate workflows for CUDA, Metal (macOS) optimized builds
- Artifact distribution: Pre-built wheels via custom index at `https://abetlen.github.io/llama-cpp-python/whl/`

**Documentation:**
- Service: ReadTheDocs (`.readthedocs.yaml` configuration present)
- Build image: Ubuntu 22.04 with Python 3.11
- Documentation site: https://llama-cpp-python.readthedocs.io/

## Environment Configuration

**Required env vars:**
- Build-time:
  - `CMAKE_ARGS` - CMake options (e.g., `-DGGML_CUDA=ON` for CUDA support)
  
- Runtime (optional, via Pydantic Settings):
  - Model loading configuration loaded from environment
  - Server configuration via `.env` files or env vars

**Optional accelerator paths:**
- `CUDA_PATH` - NVIDIA CUDA toolkit (Windows)
- `HIP_PATH` - AMD HIP toolkit (Windows)
- `LLAMA_CPP_LIB_PATH` - Custom llama library location

**Secrets location:**
- `.env` files (Pydantic Settings integration)
- Environment variables (standard practice)
- No embedded credentials in source code

## Webhooks & Callbacks

**Incoming:**
- Endpoints: None (read-only API server)
- WebSocket support: Not used
- Server-sent events: Yes, for streaming completions via `sse-starlette`

**Outgoing:**
- Webhooks: None
- Callbacks: C library callbacks via ctypes (e.g., logging callback in `_logger.py`)

## Compatible Frameworks & Libraries

**Framework Integration (Optional):**
- LangChain - Custom LLM wrapper support (`examples/high_level_api/langchain_custom_llm.py`)
- LlamaIndex - Compatible via OpenAI API drop-in replacement
- Gradio - Chat UI examples in `examples/gradio_chat/`

**Model Sources:**
- Hugging Face Model Hub (via `huggingface-hub` package)
  - Format: GGUF models (quantized)
  - Examples: Qwen, Llama, Mistral, Mixtral, Gemma models
- Local GGUF files
- OpenLLM community models

## Hardware Acceleration Options

**Available (via CMAKE_ARGS):**
- CUDA (NVIDIA GPUs) - `DGGML_CUDA=ON`
- Metal (macOS) - `DGGML_METAL=ON`
- OpenBLAS (CPU optimization) - `DGGML_BLAS=ON` with `DGGML_BLAS_VENDOR=OpenBLAS`
- VULKAN (cross-platform GPU)
- SYCL (Intel GPU)
- RPC (distributed inference)

---

*Integration audit: 2026-04-15*
