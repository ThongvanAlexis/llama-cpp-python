# Testing Patterns

**Analysis Date:** 2026-04-15

## Test Framework

**Runner:**
- pytest 7.4.0+
- Config: `pyproject.toml` with `[tool.pytest.ini_options]`
- Test paths: `testpaths = "tests"`

**Assertion Library:**
- Python standard `assert` statements
- No third-party assertion library (unittest style or pytest-assert not explicitly used)

**Run Commands:**
```bash
pytest tests                    # Run all tests
pytest tests -v               # Verbose output
pytest tests --tb=short       # Short traceback format
pytest -k test_real_model     # Run specific test by name
```

## Test File Organization

**Location:**
- Tests co-located in separate `tests/` directory, not alongside source
- Tests organized by module: `test_llama.py`, `test_llama_chat_format.py`, `test_llama_grammar.py`, `test_llama_speculative.py`

**Naming:**
- Module test files named `test_<module>.py`: `test_llama.py` tests `llama.py`
- Test functions named `test_<function_or_feature>()`: `test_llama_cpp_tokenization()`, `test_real_model()`
- Fixtures use `@pytest.fixture` decorator

**Structure:**
```
tests/
├── test_llama.py                    # Core model functionality
├── test_llama_chat_format.py        # Chat formatting
├── test_llama_grammar.py            # Grammar parsing
└── test_llama_speculative.py        # Speculative decoding
```

## Test Structure

**Suite Organization:**
```python
import pytest
import llama_cpp
from huggingface_hub import hf_hub_download

MODEL = "./vendor/llama.cpp/models/ggml-vocab-llama-spm.gguf"

def test_llama_cpp_version():
    assert llama_cpp.__version__

def test_llama_cpp_tokenization():
    llama = llama_cpp.Llama(model_path=MODEL, vocab_only=True, verbose=False)
    assert llama
    tokens = llama.tokenize(b"Hello World")
    assert tokens[0] == llama.token_bos()
```

**Patterns:**

**Setup pattern:**
- Fixtures with Hugging Face Hub downloads for real models:
```python
@pytest.fixture
def llama_cpp_model_path():
    repo_id = "lmstudio-community/Qwen3.5-0.8B-GGUF"
    filename = "Qwen3.5-0.8B-Q8_0.gguf"
    model_path = hf_hub_download(repo_id, filename)
    return model_path
```
- Local model paths for offline tests: `MODEL = "./vendor/llama.cpp/models/ggml-vocab-llama-spm.gguf"`
- Test parameters passed via fixture arguments: `def test_real_model(llama_cpp_model_path):`

**Teardown pattern:**
- Implicit via Python garbage collection
- Context managers used for resource cleanup: `contextlib.closing()` in source code (not in tests)

**Assertion pattern:**
- Direct assertions on return values:
```python
def test_llama_cpp_tokenization():
    tokens = llama.tokenize(text)
    assert tokens[0] == llama.token_bos()
    assert tokens == [1, 15043, 2787]
    detokenized = llama.detokenize(tokens)
    assert detokenized == text
```
- Dictionary key access assertions: `assert output["choices"][0]["text"] == " jumps"`
- Length assertions: `assert len(output) == 4`, `assert len(embedding) > 0`
- Type assertions: `assert llama`, `assert llama._ctx.ctx is not None`

## Mocking

**Framework:** None currently used
- Tests use real models downloaded from Hugging Face Hub
- No unittest.mock or pytest-mock fixtures observed
- C++ bindings accessed directly without mocking

**Patterns:**
- Logit processor functions tested as callables:
```python
def logit_processor_func(input_ids, logits):
    for token in tokens:
        logits[token] *= 1000
    return logits

logit_processors = llama_cpp.LogitsProcessorList([logit_processor_func])
output = model.create_completion(
    "...",
    logits_processor=logit_processors,
)
```
- No external service mocking observed

**What to Mock:**
- Hugging Face Hub downloads could be mocked for CI environments (currently using real downloads)
- Real model inference could be mocked for fast unit tests (currently using actual models)

**What NOT to Mock:**
- Core llama.cpp C bindings must not be mocked
- Tokenization logic should test against actual models
- Grammar parsing should test actual GBNF rules

## Fixtures and Factories

**Test Data:**
```python
@pytest.fixture
def llama_cpp_model_path():
    repo_id = "lmstudio-community/Qwen3.5-0.8B-GGUF"
    filename = "Qwen3.5-0.8B-Q8_0.gguf"
    model_path = hf_hub_download(repo_id, filename)
    return model_path

@pytest.fixture
def llama_cpp_embedding_model_path():
    repo_id = "CompendiumLabs/bge-small-en-v1.5-gguf"
    filename = "bge-small-en-v1.5-q4_k_m.gguf"
    model_path = hf_hub_download(repo_id, filename)
    return model_path
```

**Test Constants:**
- Local model path for offline tests: `MODEL = "./vendor/llama.cpp/models/ggml-vocab-llama-spm.gguf"`
- Prompt templates for completion tests: `"The quick brown fox jumps over the lazy dog. The quick brown fox jumps "`
- Grammar test data embedded inline: `tree = """leaf ::= "."  node ::= leaf | "(" node node ")"  root ::= node"""`

**Location:**
- Fixtures defined at module level in test files
- Test data strings defined as module constants or inline in test functions
- Hugging Face models downloaded on-demand via `hf_hub_download()` (cached by Hugging Face)

## Coverage

**Requirements:** None enforced in configuration
- No coverage configuration in `pyproject.toml`
- No pytest-cov or coverage.py configuration observed

**View Coverage:**
```bash
# Not configured - would need to add pytest-cov
pytest tests --cov=llama_cpp --cov-report=html
```

## Test Types

**Unit Tests:**
- Test individual functions in isolation: `test_llama_cpp_tokenization()`
- Scope: Tokenization, detokenization, token ID operations
- Uses local model with `vocab_only=True` for fast execution
- Example: `test_llama_cpp_version()` tests version string

**Integration Tests:**
- Test full model inference pipeline: `test_real_llama(llama_cpp_model_path)`
- Scope: Model loading, prompting, completion generation, sampling
- Uses real models from Hugging Face Hub
- Tests features: top-k/top-p sampling, temperature, grammar constraints, state saving/loading
- Example: Grammar constraint test verifies output matches expected grammar

**E2E Tests:**
- Not explicitly present
- Closest equivalent: `test_real_model()` and `test_real_llama()` test end-to-end model loading and inference
- No external API testing observed

**Speculative Decoding Tests:**
- Dedicated test file: `test_llama_speculative.py`
- Tests draft model integration with main model

## Common Patterns

**Async Testing:**
- Not used - no async/await in test suite
- Library is synchronous (no asyncio tests)

**Error Testing:**
```python
# Grammar test with error case
def test_grammar_anyof():
    sch = {
        "properties": {
            "unit": {
                "anyOf": [
                    {"enum": ["celsius", "fahrenheit"], "type": "string"},
                    {"type": "null"},
                ]
            }
        },
        "type": "object",
    }
    grammar = llama_cpp.LlamaGrammar.from_json_schema(json.dumps(sch))
    # assert grammar.grammar is not None  # (currently commented out)
```
- ValueError testing via grammar formatter: `raise_exception()` callback in Jinja2 templates

**Parameter Testing:**
- Multiple test cases with different parameter combinations:
```python
def test_real_llama(llama_cpp_model_path):
    model = llama_cpp.Llama(
        llama_cpp_model_path,
        n_ctx=32,
        n_batch=32,
        n_ubatch=32,
        n_threads=multiprocessing.cpu_count(),
        n_threads_batch=multiprocessing.cpu_count(),
        logits_all=False,
        flash_attn=True,
    )
    # Test multiple scenarios with same model instance
```

**State Testing:**
```python
# Save and restore state to verify determinism
model.set_seed(1337)
state = model.save_state()

output1 = model.create_completion(...)
number_1 = output1["choices"][0]["text"]

output2 = model.create_completion(...)
number_2 = output2["choices"][0]["text"]

model.load_state(state)
output3 = model.create_completion(...)
number_3 = output3["choices"][0]["text"]

assert number_1 != number_2  # Different with different seed context
assert number_1 == number_3  # Same after restore
```

**Chat Format Testing:**
```python
def test_mistral_instruct():
    chat_template = "{{ bos_token }}..."
    chat_formatter = jinja2.Template(chat_template)
    messages = [
        ChatCompletionRequestUserMessage(role="user", content="..."),
        ChatCompletionRequestAssistantMessage(role="assistant", content="..."),
    ]
    response = llama_chat_format.format_mistral_instruct(messages=messages)
    prompt = ("" if response.added_special else "<s>") + response.prompt
    reference = chat_formatter.render(messages=messages, ...)
    assert prompt == reference
```

---

*Testing analysis: 2026-04-15*
