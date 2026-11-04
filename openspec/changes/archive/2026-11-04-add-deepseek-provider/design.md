# Design

## Context

RRG-61 (`add-llm-providers`) gives `nar` the `LLMProvider` contract (`nar/providers/base.py`), the provider-neutral message format (`messages.py`), `AnthropicProvider`, an OpenAI-compatible base adapter `OpenAICompatibleProvider` with the subclasses `OpenRouterProvider` and `OllamaProvider` (`openai_compat.py`), retries and error translation (`retry.py`: `RetryPolicy`, `ErrorContext`, `translate_error`) and `create_provider()` (`factory.py`). `ProviderName` and `DEFAULT_MODELS` live in `nar/config/models.py`; `--provider` choices in `nar/cli/main.py` and `scripts/run_provider.py` are generated from `ProviderName`. The base adapter already sends `stream_options: {"include_usage": true}`, ignores the SDK's own retries and maps 401/403 and 404 to key and model errors. Motivation and scope: see `proposal.md`. Required behavior: see the three spec deltas.

This change is written against RRG-61 and RRG-63 as merged in `nar` (the code was read at `nar` `main`, commit `daa5eb7`, which already contains both). RRG-61 and RRG-63 are merged in `nar` already; in `nar-specs` RRG-61 is not archived yet.

External facts (DeepSeek official docs, read 2026-10-27; the page text was obtained through a summarizing fetch, so treat exact wording as unverified until the manual check in tasks.md):
- Base URL for the OpenAI format: `https://api.deepseek.com`; the docs' curl example calls `https://api.deepseek.com/chat/completions`. Whether `/v1` is also accepted is not stated, so the adapter does not use it. Source: https://api-docs.deepseek.com/
- Current model ids: `deepseek-flash` (listed as DeepSeek-V4.1-Flash) and `deepseek-v4-pro`; legacy `deepseek-v4-flash` still accepted but retired and billed at the Flash price. The docs no longer mention `deepseek-chat` or `deepseek-reasoner`; third-party sources report they were scheduled for retirement on 2026-07-24. Sources: https://api-docs.deepseek.com/quick_start/pricing, https://api-docs.deepseek.com/api/list-models, https://api-docs.deepseek.com/
- Chat completions support function tools (`tool_choice` `none`/`auto`/`required` or a named function; the last two are not supported in thinking mode) and streaming. `stream_options.include_usage = true` adds a final chunk before `[DONE]` with `usage` for the whole request. Source: https://api-docs.deepseek.com/api/create-chat-completion
- Thinking mode is on by default (effort `high`) and is controlled with `{"thinking": {"type": "enabled|disabled"}}` (passed through `extra_body` with the OpenAI SDK). When a request includes `tools`, `reasoning_content` from earlier assistant turns must be sent back in every later request, otherwise the API answers 400. Source: https://api-docs.deepseek.com/guides/thinking_mode
- Errors: 400 invalid format, 401 wrong key, 402 insufficient balance, 422 invalid parameters, 429 rate limit, 500 server error, 503 overloaded. Source: https://api-docs.deepseek.com/quick_start/error_codes

## Goals / Non-Goals

**Goals:**
- `deepseek` works everywhere another provider works: config, `--provider`, `--show-config`, `scripts/run_provider.py`, the agent loop with tools and streaming.
- Reuse the OpenAI-compatible adapter; the new code is a thin subclass plus two small, generic hooks.
- Failures the user can act on (missing key, bad key, no balance, unknown model) are one-line errors.
- Everything testable offline with mock HTTP transports.

**Non-Goals:**
- Round-tripping `reasoning_content` (see Decision 3).
- A user-configurable base URL or a generic "any OpenAI-compatible server" provider.
- Model discovery (`GET /models`) or validating the model name before the first request.
- Pricing, balance display or token cost reporting.

## Decisions

### 1. Package layout
```
src/nar/
├── config/models.py          MODIFIED  ProviderName.DEEPSEEK = "deepseek"; DEFAULT_MODELS[DEEPSEEK]
├── providers/
│   ├── openai_compat.py      MODIFIED  OpenAICompatibleProvider gains an optional extra_body
│   │                                   (sent with every request); NEW DeepSeekProvider
│   │                                   + DEEPSEEK_BASE_URL, DEEPSEEK_API_KEY_ENV
│   ├── retry.py              MODIFIED  ErrorContext gains status_hints and
│   │                                   unknown_model_pattern; translate_error uses them
│   └── factory.py            MODIFIED  API_KEY_ENVS[DEEPSEEK]; create_provider builds DeepSeekProvider
scripts/run_provider.py       MODIFIED  docstring example (choices already follow ProviderName)
README.md                     MODIFIED  provider table, env vars, DeepSeek example
tests/test_provider_openai_compat.py  MODIFIED  DeepSeek endpoint, body, tools, usage, errors
tests/test_provider_factory.py        MODIFIED  deepseek key handling and construction
tests/test_config.py                  MODIFIED  provider enum and default model
tests/test_cli.py                     MODIFIED  help text, invalid provider message, flag
```
`nar/cli/main.py` and `nar/config/loader` need no code change: the option choices and the "unsupported provider" message are built from `ProviderName`. If the message is hard-coded anywhere, the tests above reveal it and the fix is part of the same task.

### 2. `DeepSeekProvider` is a subclass, not a configurable base
```python
class DeepSeekProvider(OpenAICompatibleProvider):
    name = "deepseek"

    def __init__(self, model: str, api_key: str, **kwargs):
        super().__init__(
            model,
            base_url=DEEPSEEK_BASE_URL,            # "https://api.deepseek.com"
            api_key=api_key,
            context=ErrorContext(
                self.name, model, api_key_env=DEEPSEEK_API_KEY_ENV,
                status_hints={402: "deepseek account has insufficient balance; top up at platform.deepseek.com"},
                unknown_model_pattern="model not exist",
            ),
            extra_body={"thinking": {"type": "disabled"}},
            **kwargs,
        )
```
Same shape as `OpenRouterProvider` and `OllamaProvider`: the subclass owns endpoint, key variable and error hints. Alternative considered: make `base_url` and key variable user settings of a generic `openai-compatible` provider. Rejected: it adds config keys, a validation surface and docs for one provider, and the course requirement names concrete providers. A second subclass later costs about ten lines.

### 3. Thinking mode is disabled by default
The internal message format (RRG-61) has no field for reasoning content, and the OpenAI-compatible adapter ignores `reasoning_content` deltas. With thinking on (DeepSeek's default) and tools present, the second request of any tool loop would omit `reasoning_content` and the API would answer 400. So `DeepSeekProvider` always sends `thinking: {"type": "disabled"}` through `extra_body`. This makes `deepseek-flash` and `deepseek-v4-pro` behave like ordinary chat models with tools, which is what `nar` needs. Trade-off: no reasoning quality gain and no `deepseek-reasoner`-style behavior. Alternatives considered: (a) add `reasoning_content` to `Message`/`LLMResponse` and echo it back: correct but changes RRG-61's contract and RRG-63's `ContextManager`, which is out of scope for an S change; (b) a `thinking` config key: premature, deferred until the message format can carry reasoning. Recorded as a follow-up in Risks.

### 4. Default model: `deepseek-flash`
`DEFAULT_MODELS[DeepSeek] = "deepseek-flash"`. The docs list it as the lighter model of the current generation and the one the Quick Start uses; it supports tools and streaming and is the cheaper of the two ids. `deepseek-v4-pro` stays selectable with `--model`. Alternatives: `deepseek-v4-pro` (stronger, costlier), legacy `deepseek-chat` (no longer in the docs, reported retired). If the manual check shows `deepseek-flash` does not accept tools, fall back to `deepseek-v4-pro` and update this decision.

### 5. Usage in streaming needs no change
The base adapter already requests `stream_options.include_usage` and reads the final chunk's `usage`. DeepSeek documents the same behavior, so `Usage` is filled with no DeepSeek-specific code. A test pins it with a mocked stream whose last chunk carries `usage` and has no `choices`.

### 6. Error mapping
`ErrorContext` (in `retry.py`) gets two optional, provider-neutral fields; defaults keep Anthropic, OpenRouter and Ollama unchanged:
- `status_hints: Mapping[int, str]`: a fixed one-line message for a status code. DeepSeek sets 402. Because 402 is below 500 and not in `RETRYABLE_STATUS`, it already fails without retries; the hint only improves the message (default text would be `deepseek rejected the request (402): Insufficient Balance`).
- `unknown_model_pattern: str | None`: for statuses 400, 404 and 422, when the API message contains the pattern (case-insensitive), the error becomes `model '<model>' was not found by <provider>`, the same text 404 gives today. The DeepSeek docs do not document the unknown-model status or body, so the pattern is an assumption; tasks.md verifies it against the real API and adjusts the pattern (not the architecture) if it differs.

Existing handling stays: 401 and 403 with `api_key_env` give `deepseek rejected the API key; check DEEPSEEK_API_KEY`; 429 and 5xx (500, 503) are retried with the existing backoff and `retry_after`. 400 (including a malformed body) and 422 fall through to `deepseek rejected the request (<status>): <api message>`.

### 7. Missing key
`factory.API_KEY_ENVS` gets `ProviderName.DEEPSEEK: "DEEPSEEK_API_KEY"`; `_api_key()` already raises `ProviderError("DEEPSEEK_API_KEY is not set; export it to use the deepseek provider")` before any request. `create_provider()` replaces its final `else` (which silently means Ollama) with an explicit branch for each provider so the new name cannot fall into Ollama.

### 8. New config keys
None. `provider = "deepseek"` and `model` already exist; the only change is one more allowed provider value and its default model. The key is never read from a config file (spec: environment only).

## Risks / Trade-offs

- **Model ids drift.** DeepSeek renamed its models recently (`deepseek-chat`/`deepseek-reasoner` out, `deepseek-flash`/`deepseek-v4-pro` in). The default lives in one constant (`DEFAULT_MODELS`); users can override with `model`. The README names the date the default was verified.
- **Docs facts partly unverified.** The docs were read through a summarizing fetch and the `/v1` suffix, the unknown-model response and `deepseek-flash` tool support are not confirmed by a real call. The final manual task exercises all of them.
- **No reasoning.** Thinking is disabled, so DeepSeek's reasoning modes are unavailable. Follow-up: carry `reasoning_content` in the internal format (RRG-61/RRG-63 contract) and expose a `thinking` setting.
- **`extra_body` is sent on every request.** A future DeepSeek change rejecting the `thinking` field would break the provider; the 400 message is surfaced verbatim so the cause is visible.
- **Cost.** Real calls spend DeepSeek balance; the manual verification uses short prompts.
- **Merge order.** The spec deltas modify RRG-61 requirements, so `openspec validate` reports archive-time INFO notices until RRG-61 is archived; archive RRG-61 first.
