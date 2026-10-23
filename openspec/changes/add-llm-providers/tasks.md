# Tasks

## [specs]

- [x] 0.1 Link this change to Linear RRG-61 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-llm-providers/` on GitHub

## [nar]

## 1. Branch and dependencies

- [x] 1.1 In `nar/`, update `main` and create a branch from Linear's RRG-61 branch name; verify `git status` shows the new branch
- [x] 1.2 Add `anthropic` and `openai` to `pyproject.toml` dependencies; verify `pip install -e ".[dev]"` succeeds and `python -c "import anthropic, openai"` works

## 2. Internal format and contract

- [x] 2.1 Add `ProviderError(NarError)` with a `provider` attribute to `nar/errors.py` and a test in `tests/test_errors.py`; verify `pytest tests/test_errors.py` passes
- [x] 2.2 Implement `nar/providers/messages.py` (`ToolCall`, `Message` with `user`/`assistant`/`tool_result` constructors, `Usage`, `LLMResponse`, `StopReason`, `TextDelta`, `ResponseDone`) and `nar/providers/base.py` (`LLMProvider` with abstract `stream()` and `complete()` that drains it) from design.md §2–3, with `tests/test_provider_base.py` using a fake provider; verify `complete()` returns the same response the stream ends with

## 3. Retries and errors

- [x] 3.1 Implement `nar/providers/retry.py` (`RetryPolicy`, `RetryNotice`, retryable classification for both SDKs, `Retry-After` handling capped at 60 s, no retry after the first text delta, error translation table from design.md §6) with `tests/test_retry.py` using a fake `sleep`; verify 429→200 retries once with a notice, 503 always gives up after 4 attempts, 401 and 404 are not retried, and a failure after the first delta is not retried

## 4. Adapters

- [x] 4.1 Implement `nar/providers/anthropic.py` (message/tool conversion, consecutive tool results merged into one user message, SSE event assembly, stop reasons, usage) with `tests/test_provider_anthropic.py` using `httpx2.MockTransport`; verify text streaming, a tool call split over several `input_json_delta` events, the request body sent for a tool round trip, and 401/404 messages
- [x] 4.2 Implement `nar/providers/openai_compat.py` (`OpenAICompatibleProvider`, `OpenRouterProvider`, `OllamaProvider` with `OLLAMA_HOST`) with `tests/test_provider_openai_compat.py` using `httpx2.MockTransport`; verify text streaming, tool-call deltas accumulated by index, invalid tool JSON → `{}`, usage from the final chunk, the request body for a tool round trip, the OpenRouter base URL and headers, and the Ollama connection-refused message
- [x] 4.3 Implement `nar/providers/factory.py` (`DEFAULT_MODELS`, `create_provider(config, on_retry)` checking the API key env var before any request) and `nar/providers/__init__.py`; verify with `tests/test_provider_factory.py` that each provider name builds the right adapter and a missing `OPENROUTER_API_KEY`/`ANTHROPIC_API_KEY` raises `ProviderError` naming the variable

## 5. Configuration and CLI

- [x] 5.1 Update `nar/config/models.py` (`ProviderName` enum, `model: str | None` resolved from `DEFAULT_MODELS`) and `nar/config/loader.py` (defaults from `model_construct()` without `None` values, `overrides` layer with source `command line`); extend `tests/test_config.py` for every scenario in `specs/configuration/spec.md`; verify `pytest tests/test_config.py` passes
- [x] 5.2 Add `--provider` (Choice) and `--model` to `nar/cli/main.py`, make single mode stream the answer from `create_provider()` and print retry notices to stderr; extend `tests/test_cli.py` with a fake provider for the scenarios in `specs/cli-entrypoint/spec.md`; verify `pytest tests/test_cli.py` passes and `nar --help` lists `anthropic|openrouter|ollama`
- [x] 5.3 Add `scripts/run_provider.py` (design.md §9); verify `python scripts/run_provider.py --provider ollama --with-tool --no-stream "What is the weather in La Paz?"` prints a `get_weather` tool call and the stop reason
- [x] 5.4 Update `nar/README.md` (providers, default models, `--provider`/`--model`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`, `OLLAMA_HOST`, retries, `scripts/run_provider.py`); verify every command in the new sections runs as written

## 6. Delivery

- [ ] 6.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR referencing RRG-61

## 7. Manual verification (RRG-61 "done when")

- [ ] 7.1 Walk through RRG-61's "done when" ("the same prompt answers with the three providers changing only the config") in a fresh terminal with `NAR_HOME` pointing at a temp folder, and record what was observed for each step:
  - `nar --show-config` with no files → `provider = openrouter`, `model = openrouter/auto`
  - `.nar/config.toml` with `provider = "ollama"` and a local model → `nar "Say hello in one sentence"` streams an answer, exit 0
  - Same prompt with `provider = "openrouter"` (needs `OPENROUTER_API_KEY`) → answer, exit 0
  - Same prompt with `provider = "anthropic"` (needs `ANTHROPIC_API_KEY`) → answer, exit 0
  - `nar --provider ollama --model <local model> "hello"` overrides the file; `--show-config` shows `command line` as the source
  - Unset the API key → one-line error naming the variable, exit 1; `provider = "openai"` → error listing the three providers, exit 1
  - Ollama with an unknown model → error naming model and provider; Ollama stopped (or `OLLAMA_HOST` pointing at a closed port) → retry notices, then an error suggesting to start Ollama

  Observed on 2026-10-23 (Linux, Ollama 0.33.3 on CPU, `NAR_HOME` in a temp folder):
  - `--show-config` with no files → `provider = openrouter  (default)`, `model = openrouter/auto  (default)` ✔
  - `provider = "ollama"`, `model = "qwen3.5:4b"`: `scripts/run_provider.py` streamed "Hello there!" (stop `end`, 1558 output tokens, ~10 min on CPU) and, with `--with-tool --no-stream`, returned `get_weather {"city": "La Paz"}` with stop `tool_calls` ✔. `nar "Say hello in one sentence"` exited 0 after 35 min with an empty answer: the reasoning model spent the whole 8192-token limit thinking. Single mode now warns when the answer is cut off by the output limit ✔ (the empty answer is a model/hardware limit, not an adapter bug)
  - `openrouter` with a real key (`model = openrouter/auto`): `nar "Say hello in one sentence"` → "Hello and welcome!", 8.8 s, exit 0 ✔; `scripts/run_provider.py --with-tool --no-stream` → `get_weather {"city": "La Paz"}`, stop `tool_calls` ✔
  - `anthropic` with a real key: **pending** (no `ANTHROPIC_API_KEY` on this machine). With an invalid key, both real APIs answered and `nar` printed `openrouter rejected the API key; check OPENROUTER_API_KEY` / `anthropic rejected the API key; check ANTHROPIC_API_KEY`, exit 1, no retry ✔
  - `--provider anthropic --show-config` → `provider = anthropic  (command line)` ✔
  - Missing `OPENROUTER_API_KEY` / `ANTHROPIC_API_KEY` → one-line error naming the variable, exit 1 ✔; `provider = "openai"` in the file → error listing `anthropic, openrouter, ollama`, exit 1 ✔; `--provider openai` → Click usage error listing the three, exit 2 ✔
  - `model = "no-such-model"` on Ollama → `model 'no-such-model' was not found by ollama`, exit 1 ✔; `OLLAMA_HOST=127.0.0.1:59999` → 3 retry notices (1.2 s, 2.1 s, 4.5 s), then `cannot reach Ollama at http://127.0.0.1:59999; is it running? (start it with 'ollama serve')`, exit 1 ✔
