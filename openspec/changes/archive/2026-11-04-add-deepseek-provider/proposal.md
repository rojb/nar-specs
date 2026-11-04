# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-79
- **Video timestamps covered:** none (this change is not in the course video; it extends the course requirement "selectable model provider")
- **Depends on:** RRG-61 `add-llm-providers` (must be archived before this change is archived, because the requirements this change modifies are introduced there), RRG-63 `add-agent-loop` (the "done when" needs the agent loop to call tools)
- **Unblocks:** closes the pending agent check of RRG-64 `add-file-tools` (write and edit a file with `approval = "yolo"` through a real model)
- **Course requirement covered:** selectable model provider (adds a fourth provider next to Anthropic, OpenRouter and Ollama)

## Why

The team needs a cheap, tool-capable model to exercise the agent end to end. DeepSeek exposes an OpenAI-compatible chat completions API with tool calling and streaming, so it fits the adapter RRG-61 already built. Several pending checks (RRG-63's loop, RRG-64's write and edit tools with diff display) are waiting for a real model run; a low-cost provider unblocks them without depending on OpenRouter credits or a local GPU.

## What Changes

- Add `deepseek` as a fourth value of the `provider` setting, selectable with `provider = "deepseek"` in `~/.nar/config.toml` or `.nar/config.toml` and with `--provider deepseek` on the command line.
- Read the credential from the environment variable `DEEPSEEK_API_KEY`; a missing key is reported before any request, naming the variable.
- Send requests to DeepSeek's OpenAI-compatible endpoint by reusing RRG-61's OpenAI-compatible adapter, with a documented default model that supports tool calls and streaming.
- Keep DeepSeek's reasoning ("thinking") mode off by default so multi-step tool conversations work with `nar`'s provider-neutral message format (design explains why).
- Report DeepSeek-specific failures in one line: rejected key (401) names `DEEPSEEK_API_KEY`, insufficient balance (402) says the account has no balance, unknown model names the model and provider; rate limits (429) and server errors (500, 503) keep using the existing retries.
- Update the `nar` README (provider table, environment variables, DeepSeek example) and `scripts/run_provider.py` (docstring example; the `--provider` choices follow the new setting value).
- Add a roadmap row for this change in `docs/ROADMAP.md`.

Out of scope: sending DeepSeek's reasoning content back to the API (needs a field in the internal message format owned by RRG-61/RRG-63), DeepSeek-specific features (context caching tuning, JSON output mode, beta strict tool schemas), a configurable base URL for arbitrary OpenAI-compatible servers, and the Anthropic-style DeepSeek endpoint.

## Capabilities

### New Capabilities

<!-- None: all behavior belongs to capabilities introduced by RRG-61. -->

### Modified Capabilities

- `llm-providers`: supported providers (now four), credentials from the environment (adds `DEEPSEEK_API_KEY`), non-retryable provider errors (adds insufficient balance and DeepSeek unknown model).
- `configuration`: the `provider` setting accepts `deepseek`, with its documented default model.
- `cli-entrypoint`: the `--provider` option and its help and error text include `deepseek`.

All three capabilities are introduced by the in-flight change `add-llm-providers` (RRG-61) and are not yet in `openspec/specs/`; the deltas here modify the requirement text as RRG-61 wrote it.

## Impact

- **Code (`nar` repo):** `src/nar/config/models.py` (`ProviderName`, `DEFAULT_MODELS`), `src/nar/providers/openai_compat.py` (new `DeepSeekProvider`, small `extra_body` hook on the shared adapter), `src/nar/providers/retry.py` (insufficient-balance and unknown-model messages), `src/nar/providers/factory.py` (key variable and construction), `scripts/run_provider.py` (docstring), `README.md`, and tests under `tests/` that use mock HTTP transports like the existing provider tests (offline).
- **Dependencies:** none new (the `openai` SDK from RRG-61 is reused).
- **User-visible:** `nar --provider deepseek ...`, `provider = "deepseek"`, `DEEPSEEK_API_KEY`; `--help` and the unsupported-provider errors list one more name.
- **Merge order:** RRG-61 and RRG-63 merged in `nar` before the code branch starts; RRG-61 archived in `nar-specs` before this change is archived.
