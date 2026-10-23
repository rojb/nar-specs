# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-61
- **Video timestamps covered:** 00:05:20 (LLM Connection, OpenRouter & Streaming/Non-streaming), 01:01:09 (Retrying Requests & Exponential Backoff)
- **Depends on:** RRG-60 `scaffold-cli` (archived)
- **Runs in parallel with:** RRG-62 `add-tool-framework` (Ruben)
- **Unblocks:** RRG-63 `add-agent-loop` (together with RRG-62), RRG-70 `add-context-compaction` (summaries use the provider), RRG-78 `/model`
- **Course requirement covered:** *selectable model provider* (Anthropic, OpenRouter, local via Ollama)

## Why

`nar` can read its configuration but cannot talk to any model yet, so it cannot answer anything. The agent loop (RRG-63) needs one way to send a conversation to a model and get back text and tool calls, no matter which provider the user picked. The course also requires that the user can choose the provider (Anthropic, OpenRouter or a local Ollama model) without changing code.

## What Changes

- Add a common **LLM provider contract** with two ways to call a model: a complete response, and a streamed response that delivers text as it is generated and ends with the same complete response. Both use one **internal message format** (user, assistant, tool-result messages; assistant tool calls with id, name and arguments) and an optional system prompt.
- Add three **provider adapters**: **Anthropic** (official SDK), **OpenRouter** (OpenAI-compatible API) and **Ollama** (local, OpenAI-compatible API). Each converts the internal messages and the provider-neutral tool definitions (`name`, `description`, `parameters`, as exported by RRG-62's registry) to its API format, and converts the tool calls in the response back to the internal format.
- **Retry with exponential backoff** on rate limits (429), server errors (5xx), connection errors and timeouts, telling the user each time it retries. Other errors (bad API key, unknown model) fail at once with a clear message.
- **Choose the provider and model** from the configuration and from new command-line flags: `nar --provider ollama --model qwen2.5-coder`. The `provider` setting only accepts `anthropic`, `openrouter` and `ollama`; when `model` is not set, each provider uses its own default model. API keys come from environment variables (`ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`), never from config files.
- **Single mode answers for real:** `nar "<prompt>"` sends the prompt to the selected provider and streams the answer to the terminal (no tools, no history; RRG-63 replaces it with the agent loop). Interactive mode stays a placeholder until RRG-63.
- Add a developer script that sends a prompt to any provider, streamed or not, optionally with a sample tool, and prints the internal response, so the adapters can be checked before the agent loop exists.

Out of scope: the agent loop, conversation history and system prompt content (RRG-63), running tools (RRG-62/RRG-63), Rich rendering of the answer (RRG-63), the `/model` command (RRG-78), token counting for compaction (RRG-63/RRG-70).

## Capabilities

### New Capabilities

- `llm-providers`: the provider contract, the internal message and tool-call format, streamed and complete responses, conversion of tool definitions and tool calls for each provider, the supported providers and their credentials, retries with backoff, and provider error reporting.

### Modified Capabilities

- `configuration`: `provider` is limited to the supported providers, `model` defaults per provider, and command-line flags override the configuration files.
- `cli-entrypoint`: single mode sends the prompt to the selected model and streams the answer; new `--provider` and `--model` options.

## Impact

- **Code (`nar` repo):** new `src/nar/providers/` package (contract, internal message format, retry logic, Anthropic and OpenAI-compatible adapters, factory), a `ProviderError` added to `src/nar/errors.py`, `NarConfig` and the loader updated for the new provider rules and command-line overrides, `cli/main.py` updated for the flags and the real single mode, a `scripts/run_provider.py` developer script, README updated.
- **Dependencies:** `anthropic` and `openai` (official SDKs; `httpx` comes with them).
- **Interfaces other changes rely on:** RRG-63 calls `stream()`/`complete()` with internal messages and the registry's neutral tool definitions; RRG-70 uses `complete()` to summarize; RRG-78 switches provider/model through the same factory.
- **User-visible:** `--provider` and `--model` flags; `nar "<prompt>"` now needs a reachable provider (an API key or a running Ollama); a `provider` value other than the three supported ones is now a configuration error.
