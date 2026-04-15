# Technology Stack

**Analysis Date:** 2026-04-15

## Languages

**Primary:**
- Python 3.8+ - Core library and server implementation
  - Supported versions: 3.8, 3.9, 3.10, 3.11, 3.12, 3.13

**Secondary:**
- C/C++ - llama.cpp native library (vendored in `vendor/llama.cpp/`)
- CMake - Build system for compiling native bindings

## Runtime

**Environment:**
- Python 3.8 minimum (see `pyproject.toml` line 20)

**Package Manager:**
- pip (standard Python package manager)
- Optional: uv (for faster dependency resolution in CI/CD)
- Lockfile: Not present (uses `pyproject.toml` for dependency specification)

## Frameworks

**Core:**
- ctypes (stdlib) - Low-level C library bindings via `llama_cpp._ctypes_extensions`
- typing-extensions 4.5.0+ - Type hints for Python 3.8+ compatibility

**Server (Optional):**
- FastAPI 0.100.0+ - RESTful API server (`llama_cpp/server/app.py`)
- Uvicorn 0.22.0+ - ASGI web server
- Starlette 1.x (via FastAPI) - Web framework
- Pydantic 2.0+ (via pydantic-settings) - Data validation and settings
- Pydantic Settings 2.0.1+ - Environment configuration management
- SSE-Starlette 1.6.1+ - Server-sent events for streaming responses
- Starlette-Context 0.3.6 - Request context middleware

**Templating:**
- Jinja2 2.11.3+ - Chat format templates (`llama_cpp/llama_chat_format.py`)

**Caching:**
- DiskCache 5.6.1+ - Local file-based caching for model state

**Data Processing:**
- NumPy 1.20.0+ - Array operations for model I/O

## Key Dependencies

**Critical:**
- llama.cpp (vendored submodule in `vendor/llama.cpp/`) - Core inference engine
  - Status: Actively vendored and kept in sync via GitHub Actions
  - Integration: Compiled to shared library (`libllama.so`/`llama.dll`/`libllama.dylib`)
  - Loading: `llama_cpp._ctypes_extensions.load_shared_library()`

**Infrastructure:**
- Jinja2 2.11.3+ - Used for dynamic chat format templates
- NumPy 1.20.0+ - Numerical operations and tensor handling
- DiskCache 5.6.1+ - KV state caching for model evaluation (`llama_cpp/llama_cache.py`)

**Testing (Optional):**
- pytest 7.4.0+ - Test runner (`pyproject.toml` line 42)
- scipy 1.10+ - Scientific computing for test comparisons
- httpx 0.24.1+ - Async HTTP client for testing server endpoints

**Development (Optional):**
- ruff 0.15.7+ - Python linter and formatter (required version)
- twine 4.0.2+ - PyPI package publication
- mkdocs 1.4.3+ - Documentation generation
- mkdocstrings 0.22.0+ - API documentation from docstrings
- mkdocs-material 9.1.18+ - Material theme for documentation

## Configuration

**Environment:**
- Configuration via environment variables (for installation):
  - `CMAKE_ARGS` - CMake build options (e.g., `-DGGML_BLAS=ON`)
  - `LLAMA_CPP_LIB_PATH` - Override path to compiled llama.cpp library (`llama_cpp/llama_cpp.py` line 36)
  - `CUDA_PATH` - CUDA toolkit location (Windows, detected in `llama_cpp/_ctypes_extensions.py` line 55)
  - `HIP_PATH` - AMD HIP toolkit location (Windows, detected in `llama_cpp/_ctypes_extensions.py` line 58)

**Build:**
- `pyproject.toml` - Python project metadata and dependencies
- `CMakeLists.txt` - Native library compilation configuration
- `mkdocs.yml` - Documentation site configuration

**Server Configuration:**
- `.env` files (loaded by Pydantic Settings) - Runtime configuration
- `llama_cpp/server/settings.py` - ModelSettings and ServerSettings classes with BaseSettings validation

## Platform Requirements

**Development:**
- C compiler:
  - Linux: gcc or clang
  - Windows: Visual Studio or MinGW
  - macOS: Xcode
- CMake 3.21+ (from `CMakeLists.txt` line 1)
- scikit-build-core 0.9.2+ (build backend in `pyproject.toml` line 2)

**Production:**
- Python 3.8+
- Compiled shared library (`libllama.so` on Linux/FreeBSD, `.dylib` on macOS, `.dll` on Windows)
- Optional: CUDA 11.x+ for GPU acceleration (nvidia)
- Optional: HIP for AMD GPU acceleration
- Optional: OpenBLAS, Metal (macOS), or other accelerators (configured via CMAKE_ARGS)

## Build System Details

**Scikit-build-core Integration:**
- Wheel packages: `llama_cpp` module
- Python API: `py3` (line 66, `pyproject.toml`)
- CMake verbose output enabled
- Minimum scikit-build version: 0.5.1 (line 69, `pyproject.toml`)
- Source distribution includes: `.git` and `vendor/llama.cpp/*` (line 70, `pyproject.toml`)
- Version extracted from: `llama_cpp/__init__.py` via regex (lines 72-74, `pyproject.toml`)

## Linting & Formatting

**Ruff (0.15.7+):**
- Configuration: `pyproject.toml` lines 82-91
- Target Python: 3.8
- Line length: 88 characters
- Selected rules: E4, E7, E9 (syntax/runtime errors)
- Ignored: E712 (comparison to True/False)
- Source directories: `llama_cpp`, `tests`
- Excludes: `vendor`, `examples/notebooks`

---

*Stack analysis: 2026-04-15*
