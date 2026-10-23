# Design

## Context

`nar` (after RRG-60) has a Click command, a validated `NarConfig` with free-text `provider` and `model` keys, and a placeholder single mode. RRG-62 (in parallel) exports tool definitions as plain dicts `{name, description, parameters}`; this change consumes that shape without importing RRG-62 code, so the two changes can merge in any order. Motivation and scope: see `proposal.md`. Required behavior: see `specs/llm-providers`, `specs/configuration` and `specs/cli-entrypoint`.

Constraints:
- Python 3.11+, Windows (PowerShell) and Linux/macOS, plain `pip` (same as RRG-60).
- SDK versions available today: `anthropic` 2.x and `openai` 3.x, both built on `httpx2`.
- Team machines have Ollama installed; API keys for Anthropic and OpenRouter are personal and must never land in a repo.

## Goals / Non-Goals

**Goals:**
- One small contract (`LLMProvider`) that RRG-63 can drive without knowing which provider is behind it.
- Adapters that are testable offline (no network in `pytest`).
- One retry policy for every provider, visible to the user.

**Non-Goals:**
- Async APIs. The CLI and the agent loop are synchronous; see Decision 2.
- Prompt caching, extended thinking, images, token counting before sending. Each can be added inside an adapter later without changing the contract.
- Rich output (RRG-63). Single mode prints plain text with `click.echo`.

## Decisions

### 1. Package layout
```
src/nar/
├── errors.py                 MODIFIED  + ProviderError
├── config/
│   ├── models.py             MODIFIED  ProviderName enum, model resolved per provider
│   └── loader.py             MODIFIED  command-line override layer, unset-default handling
├── cli/main.py               MODIFIED  --provider/--model, real single mode
└── providers/                NEW
    ├── __init__.py           NEW  re-exports the public API
    ├── messages.py           NEW  Message, ToolCall, LLMResponse, StopReason, Usage, stream events
    ├── base.py               NEW  LLMProvider (abstract): complete(), stream()
    ├── retry.py              NEW  RetryPolicy, with_retries(), retryable-error classification
    ├── anthropic.py          NEW  AnthropicProvider
    ├── openai_compat.py      NEW  OpenAICompatibleProvider, OpenRouterProvider, OllamaProvider
    └── factory.py            NEW  create_provider(config, on_retry) and DEFAULT_MODELS
scripts/run_provider.py       NEW  developer runner
```

### 2. Synchronous contract
```python
class LLMProvider(ABC):
    name: str
    def complete(self, messages, *, system=None, tools=(), max_tokens=8192) -> LLMResponse
    def stream(self, messages, *, system=None, tools=(), max_tokens=8192) -> Iterator[StreamEvent]
```
`complete()` is implemented in the base class by draining `stream()`, so each adapter writes only the streaming path and both modes always agree (spec: "the stream SHALL end with the same complete response").
- **Why sync:** Click, `input()`/prompt_toolkit and the tools are synchronous; an async contract would force `asyncio.run` into every caller for no gain in a single-user CLI. **Alternative rejected:** async like the video (it uses `AsyncOpenAI`). RRG-74 (MCP, async SDK) can run its own event loop in a thread.
- **Why `complete()` over streaming:** one code path per provider instead of two parsers per provider. The non-streamed HTTP call is not needed for anything the specs require.

### 3. Internal message format (`providers/messages.py`)
Frozen dataclasses, no SDK types leak out:

| Type | Fields |
|---|---|
| `ToolCall` | `id: str`, `name: str`, `arguments: dict[str, Any]` |
| `Message` | `role: Literal["user", "assistant", "tool"]`, `content: str = ""`, `tool_calls: tuple[ToolCall, ...] = ()`, `tool_call_id: str \| None = None`, `is_error: bool = False` |
| `Usage` | `input_tokens: int`, `output_tokens: int` |
| `LLMResponse` | `content: str`, `tool_calls: tuple[ToolCall, ...]`, `stop_reason: StopReason`, `usage: Usage \| None` |
| `StopReason` | `StrEnum`: `end`, `tool_calls`, `max_tokens` |
| `TextDelta` / `ResponseDone` | stream events: `TextDelta(text)`, last event `ResponseDone(response)` |

Constructors `Message.user(text)`, `Message.assistant(text, tool_calls)`, `Message.tool_result(call_id, text, is_error=False)` keep RRG-63's code short. The system prompt is a separate argument because Anthropic takes it outside `messages`. Tool definitions are `Sequence[Mapping[str, Any]]` in RRG-62's neutral shape.

Unparseable tool arguments (OpenAI-compatible APIs send them as a JSON string) become `{}` (spec scenario); RRG-62's argument validation then reports the missing fields to the model.

### 4. Adapters
Both adapters call the HTTP API with `stream=True` and assemble the response themselves.

**Anthropic** (`anthropic.Anthropic`, `messages.create(stream=True)`):
- Tools → `{"name", "description", "input_schema": parameters}`.
- Assistant message → content blocks `text` + `tool_use(id, name, input)`. Consecutive `tool` messages → **one** `user` message with several `tool_result` blocks (`tool_use_id`, `content`, `is_error`), because the API requires every result of a turn in the next user message.
- Events: `content_block_start` (text or `tool_use` with id/name), `content_block_delta` (`text_delta` → `TextDelta`; `input_json_delta` → accumulate JSON), `message_start`/`message_delta` (usage, `stop_reason`). Stop reasons: `tool_use` → `tool_calls`, `max_tokens` → `max_tokens`, anything else → `end`.

**OpenAI-compatible** (`openai.OpenAI`, `chat.completions.create(stream=True, stream_options={"include_usage": True})`), shared by OpenRouter and Ollama:
- System prompt → first `{"role": "system"}` message. Tools → `{"type": "function", "function": {name, description, parameters}}`.
- Assistant message → `content` + `tool_calls[{id, type: "function", function: {name, arguments: json.dumps(...)}}]`; tool message → `{"role": "tool", "tool_call_id", "content"}` (errors prefixed with `Error: ` since the format has no error flag).
- Tool-call deltas are accumulated by `index` (id and name arrive first, `arguments` arrive in pieces). `finish_reason`: `tool_calls` → `tool_calls`, `length` → `max_tokens`, else `end`. If tool calls were assembled but the finish reason says otherwise (some Ollama models), the stop reason is `tool_calls`.
- `OpenRouterProvider`: base URL `https://openrouter.ai/api/v1`, key `OPENROUTER_API_KEY`, headers `X-Title: nar` and `HTTP-Referer: https://github.com/rojb/nar` (OpenRouter's app attribution).
- `OllamaProvider`: base URL `$OLLAMA_HOST/v1` (default `http://localhost:11434`; a host without scheme gets `http://`), dummy key `ollama`.

SDK clients are created with `max_retries=0` (our policy is the only retry layer) and a 120 s timeout. Adapters accept an optional `http_client` so tests inject `httpx2.MockTransport` and exercise the real SDK parsing, including SSE streams.
- **Alternative rejected:** LiteLLM (one dependency covering everything) — large, hides the conversion the course wants us to build, and its own retry/format layers would duplicate this one. **Alternative rejected:** Ollama's native `/api/chat` — the OpenAI-compatible endpoint lets two providers share one adapter.

### 5. Retries (`providers/retry.py`)
`RetryPolicy(max_retries=3, base_delay=1.0, max_delay=60.0, sleep=time.sleep)`. Delay for retry *n* (1-based) = `min(base_delay * 2**(n-1), max_delay)` plus up to 25 % random jitter → about 1 s, 2 s, 4 s. A `Retry-After` header (seconds) replaces the computed delay, capped at `max_delay`.

Retryable: HTTP 408, 409, 429, 5xx (Anthropic 529 included), `APIConnectionError`, `APITimeoutError` (both SDKs expose the same names). Everything else is raised at once.

The retry wraps the **start** of a stream: if the failure happens before the first `TextDelta` is yielded, the whole request is retried; once text has been yielded, the error is raised as `ProviderError` (spec: never show text twice). Each retry calls `on_retry(RetryNotice(provider, attempt, max_attempts, delay, reason))`; the CLI prints it to stderr in yellow: `retrying openrouter in 2.1s (rate limited, attempt 2/4)`.

### 6. Errors
`ProviderError(NarError)` with `provider: str`. Translation table (done once, in `retry.py`, for both SDKs):

| Cause | Message (one line) |
|---|---|
| Missing key (checked in the factory, before any request) | `OPENROUTER_API_KEY is not set; export it to use the openrouter provider` |
| 401 / 403 | `anthropic rejected the API key; check ANTHROPIC_API_KEY` |
| 404 | `model 'no-such-model' was not found by ollama` |
| Connection error on Ollama after retries | `cannot reach Ollama at http://localhost:11434; is it running? (start it with 'ollama serve')` |
| Retries exhausted | `openrouter failed after 4 attempts: <last error>` |
| Other 4xx | `<provider> rejected the request (<status>): <API message>` |

### 7. Configuration
`provider: ProviderName` (`StrEnum`: `anthropic`, `openrouter`, `ollama`), default `openrouter`; the loader's existing enum handling lists the allowed values in errors. `model: str | None = None`; a `model_validator(mode="after")` fills it from `DEFAULT_MODELS` when unset:

| Provider | Default model |
|---|---|
| `anthropic` | `claude-sonnet-5-5` |
| `openrouter` | `openrouter/auto` (unchanged from RRG-60) |
| `ollama` | `qwen2.5-coder` |

The loader builds its defaults layer from `NarConfig.model_construct()` dumped with `exclude_none=True`, so a setting whose default depends on another setting (only `model` today) is absent from the defaults and gets resolved after merging; `--show-config` still shows it with source `default`.

`load_config(cwd, overrides=None)`: `overrides` is one more layer merged after the project file with source `command line`. No new config keys; API keys and `OLLAMA_HOST` are environment variables only (not in TOML, so they are never committed by accident).

### 8. CLI
`--provider` is a `click.Choice` of the enum values (Click lists them in `--help` and in its usage error, exit code 2); `--model` is free text. Both feed `load_config(..., overrides=...)`. Single mode: `create_provider(config, on_retry=...)` → `provider.stream([Message.user(prompt)])` → each `TextDelta` printed with `click.echo(text, nl=False)` and flushed; a final newline at the end. No system prompt, history or tools (RRG-63). The interactive placeholder is unchanged.

### 9. Developer runner
`python scripts/run_provider.py [--provider P] [--model M] [--no-stream] [--with-tool] "prompt"` reads the same config (flags override it), prints streamed text as it arrives, then a summary: tool calls (name + JSON arguments), stop reason and usage. `--with-tool` offers a sample neutral definition `get_weather(city: string)`. Errors use the same one-line format as `nar`.

### 10. Tests
`pytest` stays offline: adapters are tested with `httpx2.MockTransport` returning recorded JSON / SSE bodies (text answer, tool call split over several deltas, invalid tool JSON, 429 then 200, 503 always, 401, 404, connection refused). Retry tests inject a fake `sleep`. CLI tests replace the factory with a fake provider. Live calls happen only in the manual checklist.

## Risks / Trade-offs

- [Default models change or get retired (`claude-sonnet-5-5`, `qwen2.5-coder`)] → They are only defaults in one table; the user overrides with `model` or `--model`.
- [OpenAI-compatible servers differ (Ollama may omit usage or stream all tool calls at once)] → Usage is optional; the accumulator handles both one-shot and incremental tool-call deltas; Ollama is covered in the manual checklist.
- [No API keys on some machines] → Offline tests cover the adapters; the manual checklist records which providers were exercised live.
- [`stream=True` everywhere means `complete()` never uses the non-streamed endpoint] → Same JSON in the end; one parser per provider is worth more than the unused endpoint.
- [Changing `provider` from `str` to an enum breaks existing configs with other values] → Only `openrouter` was meaningful before; the error lists the allowed values.

## Migration Plan

`pip install -e ".[dev]"` again to get `anthropic` and `openai`. Users with `provider` set to something other than the three names must fix it (one-line error says how). Rollback: revert the PR; config files written for this change stay valid for RRG-60 except a missing `model`, which then falls back to `openrouter/auto`.
