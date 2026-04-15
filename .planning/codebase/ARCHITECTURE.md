# Architecture

**Analysis Date:** 2026-04-15

## Pattern Overview

**Overall:** Multi-layered wrapper architecture with C/C++ bindings at the core, high-level Python interfaces, and optional FastAPI server layer for HTTP-based access.

**Key Characteristics:**
- Python ctypes bindings layer (`llama_cpp.py`) wrapping native llama.cpp C library
- High-level object-oriented wrapper (`llama.py`) providing intuitive Python API
- Composable components (context, batch, sampling, tokenizer) managing model state
- Optional FastAPI server (`server/`) providing OpenAI-compatible API endpoints
- Dependency injection pattern for model management and configuration

## Layers

**C/C++ Bindings Layer:**
- Purpose: Direct FFI interface to compiled llama.cpp shared library
- Location: `llama_cpp/llama_cpp.py`, `llama_cpp/_ctypes_extensions.py`
- Contains: ctypes function definitions, enum constants, low-level API wrappers
- Depends on: Native shared library (`lib/llama.so|.dll|.dylib`)
- Used by: `_internals.py`, `llama.py`

**Internal Model/Context Management Layer:**
- Purpose: Manage lifecycle of model, context, batch processing, sampling
- Location: `llama_cpp/_internals.py`
- Contains: `LlamaModel`, `LlamaContext`, `LlamaBatch`, `LlamaSamplingContext`, `LlamaSampler`
- Depends on: Bindings layer (`llama_cpp.py`)
- Used by: High-level `Llama` class

**High-Level API Layer:**
- Purpose: Provide user-facing Pythonic interface for text generation and chat
- Location: `llama_cpp/llama.py`
- Contains: `Llama` class with completion/chat/embedding methods, sampling logic
- Depends on: `_internals.py`, caching, tokenizers, grammar, chat format handlers
- Used by: Applications, server layer

**Supporting Components Layer:**
- Purpose: Specialized functionality (caching, tokenization, chat formatting, grammar)
- Location: `llama_cpp/llama_cache.py`, `llama_cpp/llama_tokenizer.py`, `llama_cpp/llama_chat_format.py`, `llama_cpp/llama_grammar.py`, `llama_cpp/llama_speculative.py`
- Contains: Cache implementations (RAM/disk), tokenizer wrappers, chat format handlers, grammar parsing, speculative decoding
- Depends on: High-level layer, bindings layer
- Used by: `Llama` class

**Server/API Layer:**
- Purpose: Expose model inference via HTTP endpoints with OpenAI-compatible API
- Location: `llama_cpp/server/`
- Contains: FastAPI app, routes, request/response types, server settings, error handling
- Depends on: High-level API layer (`Llama`), `LlamaProxy` for model management
- Used by: HTTP clients, external applications

**Type/Configuration Layer:**
- Purpose: Data structures and type hints for requests, responses, and settings
- Location: `llama_cpp/llama_types.py`, `llama_cpp/server/types.py`, `llama_cpp/server/settings.py`
- Contains: TypedDict definitions for OpenAI-compatible types, Pydantic models for configuration
- Depends on: None (base types)
- Used by: All layers

## Data Flow

**Inference Flow (Completion):**

1. User calls `Llama.__call__(prompt, max_tokens, ...)` or `create_completion(...)`
2. Prompt is tokenized via `LlamaTokenizer.encode(text)` → token sequence
3. Token sequence validated against context window (`n_ctx`)
4. `LlamaModel.tokenize()` → list of token IDs using vocab
5. `LlamaBatch` accumulates tokens for processing
6. `LlamaContext.eval(batch)` invokes llama_cpp inference kernel
7. Model generates logits for next token
8. `LlamaSamplingContext` applies sampling (temperature, top-p, grammar constraints, etc.)
9. New token sampled from logits distribution
10. Process repeats until max_tokens or stop sequence reached
11. Generated tokens detokenized via `LlamaModel.detokenize()` → text
12. Results wrapped in `CreateCompletionResponse` TypedDict
13. If streaming, tokens yielded as `CreateCompletionStreamResponse`

**Chat Completion Flow:**

1. User calls `Llama.create_chat_completion(messages, ...)`
2. `LlamaChatCompletionHandler` (if specified) formats messages to prompt
   - E.g., `Llava15ChatHandler` for vision, `Llama3ChatHandler` for chat template
3. Handler returns formatted prompt string
4. Falls back to inference flow above
5. Response tokens formatted as `ChatCompletionMessage` with role/content

**Server Request Flow:**

1. HTTP request arrives at FastAPI endpoint (e.g., `/v1/completions`)
2. Request body parsed into `CreateCompletionRequest`
3. `prepare_request_resources()` resolves:
   - Model instance via `LlamaProxy(model_name)`
   - Grammar from string to `LlamaGrammar`
   - Logits processors/stopping criteria
4. `get_event_publisher()` creates event stream sender
5. Model inference called with prepared kwargs
6. Results published to stream channel
7. `EventSourceResponse` yields server-sent events to client

**State Management:**

- **Model State:** `LlamaModel` owns native llama_model pointer, freed on close
- **Context State:** `LlamaContext` owns native llama_context pointer, batch state
- **Token State:** Accumulated in `input_ids` numpy array, tokens tracked per sequence
- **KV Cache State:** Managed by native context, reused across calls if cache enabled
- **Sampling State:** `LlamaSamplingContext` holds temperature, sampler chain
- **Cache Management:** Optional `LlamaCache` (RAM or disk) stores full model state by token prefix

## Key Abstractions

**Llama:**
- Purpose: Main user-facing interface encapsulating model lifecycle
- Examples: `llama_cpp/llama.py:Llama`
- Pattern: Singleton-like per model (can load multiple via proxy), manages all sub-components

**LlamaProxy:**
- Purpose: Manage multiple model instances with lazy loading/switching
- Examples: `llama_cpp/server/model.py:LlamaProxy`
- Pattern: Factory/proxy pattern, tracks current model, handles model switching with cleanup

**Chat Completion Handlers:**
- Purpose: Format multi-modal chat messages to compatible prompt
- Examples: `Llava15ChatHandler`, `Llama3ChatHandler`, `Llava16ChatHandler`
- Pattern: Protocol/interface-based, registry of handlers by name

**Cache Interface:**
- Purpose: Abstract storage of model state for KV cache reuse
- Examples: `BaseLlamaCache`, `LlamaRAMCache`, `LlamaDiskCache`
- Pattern: Abstract base class with multiple implementations

**Tokenizers:**
- Purpose: Convert text ↔ token sequences
- Examples: `BaseLlamaTokenizer`, `LlamaTokenizer`
- Pattern: Abstract with fallback to llama.cpp native tokenizer

## Entry Points

**Python Library:**
- Location: `llama_cpp/__init__.py`
- Triggers: `from llama_cpp import Llama` or `import llama_cpp`
- Responsibilities: Exports main API (Llama, types, utilities)

**Direct Model Usage:**
- Location: Application code calls `Llama(model_path="...").create_completion(...)`
- Triggers: User code instantiation
- Responsibilities: Model loading, inference, streaming results

**Server Entry Point:**
- Location: `llama_cpp/server/__main__.py:main()`
- Triggers: `python -m llama_cpp.server` or `uvicorn llama_cpp.server.app:create_app`
- Responsibilities: Parse CLI args, load config, start HTTP server

**FastAPI App Factory:**
- Location: `llama_cpp/server/app.py:create_app()`
- Triggers: Server startup or programmatic app creation
- Responsibilities: Initialize FastAPI with middleware, routes, model proxy

## Error Handling

**Strategy:** Exceptions propagated up with descriptive messages, server endpoints return HTTP 400/422 for validation errors and 503 for service unavailable.

**Patterns:**
- Model validation errors (Pydantic) → HTTP 422 Unprocessable Entity
- Missing model file → ValueError with path
- Failed tokenization → RuntimeError with details
- Service unavailable (model not loaded) → HTTP 503 with error detail
- Grammar parsing errors → Exception with parse details
- Chat format not found → `LlamaChatCompletionHandlerNotFoundException`

## Cross-Cutting Concerns

**Logging:** 
- Controlled via `_logger.py:set_verbose()` flag
- Verbose mode outputs to stderr
- Server has configurable `verbose` per model

**Validation:** 
- Token count vs context window checked before inference
- Chat format selection validated against registry
- Model path existence validated before loading
- Request schemas validated by Pydantic

**Authentication:** 
- Server supports optional HTTP Bearer token via `--api-key` (in settings)
- No auth required for local use

---

*Architecture analysis: 2026-04-15*
