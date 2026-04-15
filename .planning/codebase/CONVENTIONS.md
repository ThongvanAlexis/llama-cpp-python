# Coding Conventions

**Analysis Date:** 2026-04-15

## Naming Patterns

**Files:**
- Lowercase with underscores: `llama.py`, `llama_types.py`, `llama_chat_format.py`
- Private/internal modules prefixed with underscore: `_logger.py`, `_internals.py`, `_utils.py`, `_ctypes_extensions.py`
- Server-specific code in `server/` subdirectory: `server/app.py`, `server/cli.py`, `server/errors.py`

**Classes:**
- PascalCase: `Llama`, `LlamaGrammar`, `LlamaCache`, `LlamaState`, `BaseLlamaCache`, `LlamaRAMCache`
- Abstract base classes prefixed with "Base": `BaseLlamaCache`, `BaseLlamaTokenizer`
- Private class variables use double underscore: `__backend_initialized` in `Llama`

**Functions and Methods:**
- snake_case: `create_completion()`, `tokenize()`, `detokenize()`, `set_cache()`, `_init_sampler()`
- Private methods prefixed with underscore: `_create_completion()`, `_input_ids`, `_scores`, `_find_longest_prefix_key()`
- Properties use `@property` decorator and snake_case: `cache_size`, `ctx`, `model`, `eval_tokens`

**Variables:**
- snake_case: `model_path`, `n_gpu_layers`, `n_threads`, `tensor_split`
- Prefixed with underscore for internal state: `_model`, `_ctx`, `_stack`, `_seed`, `_logits_all`
- Configuration parameters use abbreviated names matching C++ underlying API: `n_ctx`, `n_batch`, `n_threads`, `n_ubatch`

**Types and TypedDicts:**
- PascalCase: `CreateCompletionResponse`, `CompletionChoice`, `ChatCompletionResponseMessage`, `EmbeddingUsage`
- TypedDict suffix pattern: Response, Message, Choice types follow OpenAI API naming: `CreateEmbeddingResponse`, `CreateChatCompletionResponse`

## Code Style

**Formatting:**
- Line length: 88 characters (configured in `pyproject.toml`)
- Tool: Ruff (configured with target Python 3.8+)

**Linting:**
- Tool: Ruff
- Configuration in `pyproject.toml`: `[tool.ruff]`
- Rules selected: "E4", "E7", "E9" (syntax errors)
- Ignored rules: "E712" (comparison to True/False/None)

**Import Organization:**
- Standard library imports first: `os`, `sys`, `time`, `json`, `ctypes`, `typing`, `contextlib`
- Third-party library imports: `numpy`, `typing_extensions`, `fastapi`, `pydantic_settings`, `diskcache`
- Local imports: Relative imports using dot notation: `from .llama_types import *`, `from .llama_cache import BaseLlamaCache`
- Type imports from `typing` module: `List`, `Dict`, `Optional`, `Union`, `Literal`, `TypedDict`
- Future annotations enabled in many files: `from __future__ import annotations`

**Path Aliases:**
- No path aliases configured. Direct relative imports used throughout.
- Import paths respect module structure: `import llama_cpp.llama_cpp as llama_cpp`, `import llama_cpp.llama_chat_format as llama_chat_format`

## Error Handling

**Exception Strategy:**
- Raise specific exception types: `ValueError`, `RuntimeError`, `KeyError`, `ImportError`
- Error messages are descriptive and contextual
- Custom exceptions defined in `server/errors.py`: `ErrorResponse` TypedDict used for OpenAI-compatible error responses

**Patterns:**
- Validation at initialization: Model path existence checked in `Llama.__init__()` with `ValueError`
- Configuration validation: `kv_overrides` type checking with explicit `ValueError` for unknown types
- File operations wrapped in try/except: `ImportError` caught when optional dependencies (e.g., openai) are missing
- Cache key lookups raise `KeyError("Key not found")` in `llama_cache.py`
- Chat format handler lookups raise `ValueError` with descriptive messages in `llama_chat_format.py`

**Server Error Handling:**
- Custom `RouteErrorHandler` in `server/errors.py` wraps exceptions in OpenAI-style error responses
- Pattern matching on error message strings to categorize and format errors
- Unmatched exceptions logged to stderr with traceback and wrapped as 500 Internal Server Error

## Logging

**Framework:** Python standard `logging` module

**Custom Logger:**
- Module logger in `_logger.py`: `logger = logging.getLogger("llama-cpp-python")`
- Logger bridges C++ llama.cpp logs to Python logging
- C++ log levels mapped to Python logging levels via `GGML_LOG_LEVEL_TO_LOGGING_LEVEL` dict

**Patterns:**
- Verbose output controlled by `set_verbose()` function
- Debug level logging via `logger.debug()` in `llama_chat_format.py`
- Most logging suppressed through `_utils.suppress_stdout_stderr()` context manager
- stderr used for warnings and errors: `print(text.decode("utf-8"), end="", flush=True, file=sys.stderr)`

## Comments

**When to Comment:**
- Complex ctypes operations include explanatory comments: `# Type conversion and expand the list to the LLAMA_MAX_DEVICES`, `# keep a reference to the array so it is not gc'd`
- Workarounds and temporary solutions: `# TODO:` comments mark unresolved issues
- Implementation notes on non-obvious logic: `# NOTE: This is probably broken` on recarray construction

**Pattern Examples:**
- `# TODO: Make this a constant` - identifies magic numbers needing extraction
- `# NOTE: Correctly implement continue previous log` - documents incomplete implementation
- `# type: ignore` - used for type checker suppression when necessary

**Docstrings:**
- Google-style docstrings with triple quotes on public methods
- Multi-line docstrings include: Examples, Args, Returns, Raises sections
- Method docstrings follow immediate after def line
- Example in docstring: `llama_cpp.Llama(model_path="path/to/model")`

## Function Design

**Size:**
- Methods vary widely: Small property getters (5 lines), large initialization methods (400+ lines)
- Complex methods like `_create_completion()` contain nested functions for logit processing
- Aim is functional completeness rather than strict line limits

**Parameters:**
- Keyword-only parameters after `*` in large parameter lists: `def __init__(self, model_path: str, *, n_gpu_layers: int = 0, ...)`
- Parameters grouped by logical sections: Model Params, Context Params, Sampling Params
- Type hints used throughout: `Optional[List[float]]`, `Union[bool, int]`, `Dict[str, Union[bool, int, float, str]]`
- Default values provided for optional parameters

**Return Values:**
- TypedDict returns for API-compatible responses: `CreateCompletionResponse`, `CreateChatCompletionResponse`
- Generator returns for streaming: `Generator[CreateCompletionChunk, None, None]`
- Optional returns: `Optional[BaseLlamaCache]`, `Optional[int]`
- State objects returned as dataclasses: `LlamaState`

## Module Design

**Exports:**
- `llama_cpp/__init__.py` exports public API: `from .llama_cpp import *`, `from .llama import *`
- All public classes and types exported at package level
- Internal modules (prefixed with `_`) not exposed in `__init__.py`

**Barrel Files:**
- Single level of import re-exporting in `__init__.py`
- Subpackages export via `__init__.py`: `server/__init__.py` minimal
- Type imports via `from .llama_types import *` pattern used

**Organization:**
- Core inference in `llama.py` (Llama class, sampling, completion)
- Type definitions in `llama_types.py` (OpenAI-compatible TypedDict definitions)
- Chat formatting in `llama_chat_format.py` (4000+ lines with multiple formatter implementations)
- C++ bindings in `llama_cpp.py` (4900+ lines of ctypes definitions)
- Grammar support in `llama_grammar.py` (GBNF grammar parser)
- Tokenizer in `llama_tokenizer.py`
- Cache implementations in `llama_cache.py` (RAM and disk-based)
- Internals (low-level ctypes wrappers) in `_internals.py`

---

*Convention analysis: 2026-04-15*
