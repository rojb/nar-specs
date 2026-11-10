# Design

## Context

RRG-86 added `LLMProvider.check()`. Each adapter already reads its model list inside `check`: `client.models.list()` for Anthropic (only on a 404) and for the OpenAI-compatible adapters. Slash commands are registered in `src/nar/cli/commands.py` and answer on standard error through `_line`.

## Decisions

### 1. `LLMProvider.list_models()`
`src/nar/providers/base.py` (MODIFIED): a new method `list_models() -> list[str]`. The base implementation raises `ProviderError("<name> cannot list its models")`, so a provider that cannot list its models says so instead of returning an empty list.
- `AnthropicProvider` (MODIFIED) returns the ids from `client.models.list()`. Its `check` uses `list_models` for the suggestions.
- `OpenAICompatibleProvider` (MODIFIED) returns the ids from `client.models.list()`, and `check` now uses `list_models`. OpenRouter keeps its `/key` check in `check` only, since its list is public.

Both go through `call_once` with `CHECK_TIMEOUT`, as `check` does.

### 2. `/models`
`src/nar/cli/runner.py` (MODIFIED): `list_models(agent, view, spec) -> tuple[str, list[str]]`, which returns the provider's name and its model names.
- With no provider, it uses `agent.provider`.
- With a provider, it builds one with `factory.create_provider(NarConfig(provider=...))` and does not assign it. A missing key raises the factory's usual error.
- It prints `listing <provider> models...` while it waits.

`src/nar/cli/commands.py` (MODIFIED) registers `models` with the summary `list a provider's models: /models [provider] [filter]`. The command does the following:
- It sorts the names and keeps only those that contain the filter, ignoring case.
- It prints a header `<provider>: <n> models` or `<provider>: <n> of <total> models match '<text>'`.
- It prints one name per line, indented. The current model gets `  ← in use` when the listed provider is the one in use.
- `NarError` prints `error: <message>`, and `KeyboardInterrupt` prints `listing cancelled`.

No new configuration keys.

## Risks / Trade-offs

- **OpenRouter lists hundreds of models** → the filter argument, and the header tells how many there are.
- **The list a server reports may not include every alias** (for example, Anthropic's short aliases) → `/model` still accepts aliases, because its check uses `models.retrieve`.

## Migration Plan

None.

## Open Questions

None.
