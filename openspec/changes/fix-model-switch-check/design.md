# Design

## Context

`switch_model` (`src/nar/cli/runner.py`) builds the new provider with `factory.create_provider`, which fails only when the key variable is missing, and then assigns it to the agent and the compactor (RRG-81). Nothing talks to the provider until the next request, which goes through `stream_with_retries`: four attempts with backoff, so an unreachable server costs about 8 seconds per request. Every SDK already exposes a model endpoint that costs no tokens.

## Decisions

### 1. `LLMProvider.check()`
`src/nar/providers/base.py` (MODIFIED): a new method `check() -> None` that raises `ProviderError` when the provider cannot be used. The base implementation does nothing, so test doubles and any future adapter that cannot check are trusted, as today. A shared helper `unknown_model_error(provider, model, available)` builds `model '<m>' was not found by <p>` and adds `; did you mean a, b, c?` with `difflib.get_close_matches` (up to three, cutoff 0.6).

Each check makes **one attempt** with `CHECK_TIMEOUT = 10.0` seconds through `client.with_options(timeout=CHECK_TIMEOUT)`, and turns SDK errors into messages with the existing `translate_error(exc, context)`. It does not use `stream_with_retries`: the check must fail fast, and the user can retry `/model`. **Alternative rejected:** a one-token chat request. It also checks the model, but it spends tokens and is slower.

### 2. Per adapter
- `AnthropicProvider.check` (`src/nar/providers/anthropic.py`, MODIFIED): `client.models.retrieve(model)`. It accepts aliases, which the list does not include. On 404 it reads `client.models.list()` only to suggest names.
- `OpenAICompatibleProvider.check` (`src/nar/providers/openai_compat.py`, MODIFIED): `client.models.list()`, then compares ids with `self.model` through `_model_ids_match`.
- `OpenRouterProvider.check` (MODIFIED): OpenRouter's model list is public and accepts any key, so it first calls `client.get("/key", cast_to=object)` to check the key, then the list.
- `OllamaProvider.check` (MODIFIED): the same list (Ollama's `/v1/models`). A model without a tag matches `<model>:latest`, and a missing model gives `model '<m>' is not installed in Ollama; run 'ollama pull <m>'`. A connection error already maps to the existing `unreachable_hint`.

### 3. `/model` uses the check
`src/nar/cli/runner.py` (MODIFIED): `switch_model` creates the provider, prints `checking <provider> · <model>...` with `view.note`, calls `provider.check()`, and assigns the agent's and the compactor's provider only after it passes. On `ProviderError` it raises `NarError(f"{message}; still using {old.name} · {old.model}")`.

`src/nar/cli/commands.py` (MODIFIED): `_model` also catches `KeyboardInterrupt` and prints `model switch cancelled; still using <provider> · <model>`, so Ctrl+C does not leave the REPL.

No new configuration keys.

## Risks / Trade-offs

- **A provider's list may miss a model that works** (an alias or a new name) → Anthropic uses `retrieve`, which resolves aliases. For OpenAI-compatible servers, the list is what the server itself reports. The message names the model, so the user can see the mismatch.
- **One more request per `/model`** → it costs no tokens, and `/model` is rare.
- **The key can still be revoked later** → the check only covers the moment of the switch. A later failure keeps today's behavior.

## Migration Plan

None.

## Open Questions

None.
