# Codebase Structure

**Analysis Date:** 2026-04-15

## Directory Layout

```
llama-cpp-python/
├── llama_cpp/              # Main Python package
│   ├── __init__.py         # Public API exports
│   ├── llama.py            # High-level Llama class (2500+ lines)
│   ├── llama_cpp.py        # ctypes bindings to native library
│   ├── llama_types.py      # OpenAI-compatible TypedDict definitions
│   ├── llama_grammar.py    # Grammar parsing and GBNF rules
│   ├── llama_cache.py      # Cache implementations (RAM/disk)
│   ├── llama_tokenizer.py  # Tokenizer wrapper/interface
│   ├── llama_chat_format.py # Chat format handlers (Llava, Llama2, Llama3, etc.)
│   ├── llama_speculative.py # Speculative decoding (draft models)
│   ├── llava_cpp.py        # LLaVA vision model wrapper
│   ├── mtmd_cpp.py         # Multi-modal text decoder
│   ├── _internals.py       # Low-level model/context/batch management
│   ├── _ctypes_extensions.py # FFI utilities and type helpers
│   ├── _logger.py          # Logging configuration
│   ├── _utils.py           # Utility functions
│   ├── lib/                # Compiled native library (llama.so|.dll|.dylib)
│   └── server/             # Optional HTTP server (FastAPI)
│       ├── __init__.py
│       ├── __main__.py     # CLI entry point
│       ├── app.py          # FastAPI application factory
│       ├── model.py        # LlamaProxy for model management
│       ├── settings.py     # Pydantic settings for server/models
│       ├── types.py        # Request/response type definitions
│       ├── cli.py          # CLI argument parsing
│       └── errors.py       # Error handling middleware
├── tests/                  # Test suite
│   ├── test_llama.py
│   ├── test_llama_chat_format.py
│   ├── test_llama_grammar.py
│   └── test_llama_speculative.py
├── examples/               # Usage examples
│   ├── high_level_api/     # Basic usage examples
│   ├── gradio_chat/        # Gradio UI wrapper
│   ├── batch-processing/   # Batch inference example
│   └── hf_pull/            # Hugging Face model download
├── docs/                   # Documentation source
├── vendor/                 # Submodules (llama.cpp, gguf-py)
├── scripts/                # Build and utility scripts
├── pyproject.toml          # Project metadata and dependencies
├── CMakeLists.txt          # Build configuration
├── Makefile                # Common build targets
└── README.md               # Main documentation
```

## Directory Purposes

**`llama_cpp/`:**
- Purpose: Main Python package containing all implementation
- Contains: Core model wrapper, type definitions, optional server
- Key files: `llama.py` (main API), `llama_cpp.py` (bindings), `_internals.py` (lifecycle management)

**`llama_cpp/server/`:**
- Purpose: FastAPI HTTP server providing OpenAI-compatible endpoints
- Contains: Route handlers, request/response schemas, model proxy, configuration
- Key files: `app.py` (FastAPI factory), `model.py` (LlamaProxy), `settings.py` (Pydantic config)

**`llama_cpp/lib/`:**
- Purpose: Compiled native llama.cpp library
- Generated: Yes (compiled during build)
- Committed: No (built from vendor/llama.cpp via CMake)

**`tests/`:**
- Purpose: Test suite for all components
- Contains: Unit tests using pytest
- Key files: `test_llama.py` (core functionality), `test_llama_chat_format.py` (multi-modal)

**`examples/`:**
- Purpose: Demonstrations of library usage
- Contains: Standalone scripts showing inference, chat, embeddings, batch processing
- Key files: `high_level_api/high_level_api_inference.py` (basic usage template)

**`vendor/`:**
- Purpose: External dependencies (submodules)
- Contains: `llama.cpp` C++ project, `gguf-py` for GGUF file handling
- Generated: No (Git submodules)
- Committed: No (submodule pointers only)

## Key File Locations

**Entry Points:**

- `llama_cpp/__init__.py`: Main package init, exports `Llama` class and utilities
- `llama_cpp/server/__main__.py`: CLI server launcher, parses args and starts uvicorn
- `llama_cpp/server/app.py`: FastAPI app factory, called by uvicorn or programmatically

**Configuration:**

- `pyproject.toml`: Package metadata, dependencies, build config
- `CMakeLists.txt`: Native build configuration (scikit-build-core)
- `llama_cpp/server/settings.py`: Server settings (Pydantic `BaseSettings`)

**Core Logic:**

- `llama_cpp/llama.py`: Main `Llama` class implementing completion/chat/embedding
- `llama_cpp/llama_cpp.py`: ctypes bindings to native llama.cpp library
- `llama_cpp/_internals.py`: Lower-level model/context/batch management classes

**Type Definitions:**

- `llama_cpp/llama_types.py`: OpenAI-compatible TypedDict types (Completion, ChatCompletion, Embedding)
- `llama_cpp/server/types.py`: Extended server request/response types

**Supporting Modules:**

- `llama_cpp/llama_cache.py`: Caching implementations (RAM/disk)
- `llama_cpp/llama_tokenizer.py`: Tokenizer interface and wrappers
- `llama_cpp/llama_chat_format.py`: Chat format handlers (Llava, Llama2, Llama3, Phi, etc.)
- `llama_cpp/llama_grammar.py`: Grammar parser and GBNF rule definitions
- `llama_cpp/llama_speculative.py`: Speculative decoding draft models

**Server Components:**

- `llama_cpp/server/model.py`: `LlamaProxy` class for managing multiple models
- `llama_cpp/server/app.py`: Route definitions and event streaming
- `llama_cpp/server/cli.py`: Argument parsing utilities
- `llama_cpp/server/errors.py`: Custom error handlers and route wrappers

## Naming Conventions

**Files:**

- Core modules: `llama_*.py` (e.g., `llama_cache.py`, `llama_grammar.py`)
- Internal utilities: `_*.py` (e.g., `_internals.py`, `_ctypes_extensions.py`)
- Test files: `test_*.py` located in `tests/` directory

**Directories:**

- Package: lowercase underscore-separated (e.g., `llama_cpp`)
- Package subdirectories: singular purpose name lowercase (e.g., `server`)

**Classes:**

- Main classes: PascalCase (e.g., `Llama`, `LlamaModel`, `LlamaContext`, `LlamaProxy`)
- Handler classes: `Llava15ChatHandler`, `Llama3ChatHandler` (specific multi-modal types)
- Abstract/Base classes: Prefix `Base` (e.g., `BaseLlamaCache`, `BaseLlamaTokenizer`)
- Exception classes: Suffix `Error` or `Exception` (e.g., `LlamaChatCompletionHandlerNotFoundException`)

**Functions:**

- Public methods: camelCase (e.g., `create_completion`, `tokenize`, `eval`)
- Private methods: leading underscore (e.g., `_find_longest_prefix_key`, `_warn_deprecated`)

**Type Annotations:**

- TypedDict classes: Suffix `Dict` or `Response` (e.g., `CreateCompletionResponse`, `CompletionUsage`)
- Type aliases: UPPERCASE (e.g., `JsonType`)

## Where to Add New Code

**New Inference Feature (e.g., new sampling strategy):**
- Primary code: `llama_cpp/_internals.py` (add to `LlamaSamplingContext`)
- Integration: `llama_cpp/llama.py` (add parameter to `__call__` and `create_completion`)
- Tests: `tests/test_llama.py`

**New Chat Format Handler:**
- Implementation: `llama_cpp/llama_chat_format.py` (create class extending `LlamaChatCompletionHandler`)
- Registration: Add to `LlamaChatCompletionHandlerRegistry.register()`
- Tests: `tests/test_llama_chat_format.py`

**New Server Endpoint:**
- Route: `llama_cpp/server/app.py` (add to router using `@router.post(...)`)
- Types: `llama_cpp/server/types.py` (request/response schemas)
- Error handling: `llama_cpp/server/errors.py` (custom exceptions if needed)
- Integration: Call `get_llama_proxy()` to access model

**New Cache Implementation:**
- Interface: Extend `BaseLlamaCache` in `llama_cpp/llama_cache.py`
- Methods: Implement `__getitem__`, `__setitem__`, `__contains__`, `cache_size` property

**Utility Functions:**
- General helpers: `llama_cpp/_utils.py`
- Type/ctypes helpers: `llama_cpp/_ctypes_extensions.py`

## Special Directories

**`llama_cpp/lib/`:**
- Purpose: Holds compiled native library bindings
- Generated: Yes (created during pip install via scikit-build-core)
- Committed: No (output of build process)
- Loaded by: `llama_cpp/llama_cpp.py:load_shared_library()` using ctypes

**`vendor/llama.cpp/`:**
- Purpose: Upstream llama.cpp C++ project as git submodule
- Generated: No (git submodule)
- Committed: Pointers only (.gitmodules)
- Built with: CMakeLists.txt configuration

## Import Patterns

**High-level API:**
```python
from llama_cpp import Llama
llm = Llama(model_path="model.gguf")
completion = llm("prompt")
```

**Type imports:**
```python
from llama_cpp.llama_types import CreateCompletionResponse, ChatCompletionMessage
```

**Server direct use:**
```python
from llama_cpp.server.app import create_app
from llama_cpp.server.settings import ServerSettings, ModelSettings
app = create_app(server_settings=..., model_settings=...)
```

**Internal structure:**
- `llama.py` imports from `_internals`, `llama_cache`, `llama_grammar`, etc.
- `llama_cpp.py` imports from `_ctypes_extensions`
- Server imports from main package (`import llama_cpp`)

---

*Structure analysis: 2026-04-15*
