# Spec Delta

## MODIFIED Requirements

### Requirement: Supported providers
`nar` SHALL support four LLM providers, selected by name: `anthropic` (Anthropic API), `openrouter` (OpenRouter API), `ollama` (a local Ollama server) and `deepseek` (DeepSeek API). The same conversation SHALL be sendable to any of them without changes outside the provider and model selection, including conversations in which the model calls tools over several turns.

#### Scenario: Same prompt on every provider
- **WHEN** the same prompt is sent once with `provider = "anthropic"`, once with `provider = "openrouter"`, once with `provider = "ollama"` and once with `provider = "deepseek"`, each with a valid model
- **THEN** each provider returns a text answer and nothing else in the request or the code had to change

#### Scenario: Multi-turn tool conversation on DeepSeek
- **WHEN** with `provider = "deepseek"` the model answers a prompt with a tool call, the tool result is sent back, and the model then calls a second tool and finally answers in text
- **THEN** every request in the conversation is accepted by the provider and the final answer refers to the tool results

### Requirement: Credentials from the environment
API keys SHALL be read only from environment variables: `ANTHROPIC_API_KEY` for `anthropic`, `OPENROUTER_API_KEY` for `openrouter` and `DEEPSEEK_API_KEY` for `deepseek`. `ollama` SHALL need no key and SHALL use the server at `OLLAMA_HOST` when set, or `http://localhost:11434` otherwise. A missing key SHALL be reported before any request is sent.

#### Scenario: Missing API key
- **WHEN** `provider = "openrouter"` is selected and `OPENROUTER_API_KEY` is not set
- **THEN** `nar` exits with code 1 and a one-line error naming `OPENROUTER_API_KEY`, without contacting OpenRouter

#### Scenario: Missing DeepSeek API key
- **WHEN** `provider = "deepseek"` is selected and `DEEPSEEK_API_KEY` is not set or is empty
- **THEN** `nar` exits with code 1 and a one-line error naming `DEEPSEEK_API_KEY`, without contacting DeepSeek

#### Scenario: Custom Ollama host
- **WHEN** `provider = "ollama"` is selected and `OLLAMA_HOST` is `http://192.168.1.20:11434`
- **THEN** requests go to that server

### Requirement: Non-retryable provider errors
Errors that retrying cannot fix SHALL fail at once with a one-line message: rejected credentials SHALL name the environment variable to check, an unknown model SHALL name the model and provider, an account without balance (DeepSeek, HTTP 402) SHALL say that the balance is insufficient and name the provider, and an unreachable Ollama server SHALL name the address and suggest starting Ollama.

#### Scenario: Invalid API key
- **WHEN** `ANTHROPIC_API_KEY` is set to a key the API rejects
- **THEN** `nar` makes no retry, exits with code 1 and the error names `ANTHROPIC_API_KEY`

#### Scenario: Invalid DeepSeek API key
- **WHEN** `provider = "deepseek"` is selected and `DEEPSEEK_API_KEY` is set to a key DeepSeek rejects with HTTP 401
- **THEN** `nar` makes no retry, exits with code 1 and the error names `DEEPSEEK_API_KEY`

#### Scenario: Unknown model
- **WHEN** `provider = "ollama"` and `model = "no-such-model"` are selected and the server does not have that model
- **THEN** `nar` exits with code 1 and the error names `no-such-model` and `ollama`

#### Scenario: Unknown DeepSeek model
- **WHEN** `provider = "deepseek"` and `model = "no-such-model"` are selected and DeepSeek reports that the model does not exist
- **THEN** `nar` makes no retry, exits with code 1 and the error names `no-such-model` and `deepseek`

#### Scenario: DeepSeek account without balance
- **WHEN** `provider = "deepseek"` is selected and DeepSeek answers HTTP 402 (insufficient balance)
- **THEN** `nar` makes no retry, exits with code 1 and a one-line error says that the `deepseek` account has insufficient balance

#### Scenario: Ollama not running
- **WHEN** `provider = "ollama"` is selected and nothing listens at the Ollama address
- **THEN** after the retries `nar` exits with code 1 and the error names the address and suggests starting Ollama

## ADDED Requirements

### Requirement: DeepSeek provider
With `provider = "deepseek"`, `nar` SHALL send conversations to the DeepSeek API using the model from the configuration, with a documented default model that supports tool calling and streaming. Streamed responses SHALL deliver text incrementally, assemble tool calls and report token usage like the other providers. Rate limits (HTTP 429) and server errors (HTTP 500 and 503) SHALL be retried as described in the retry requirement.

#### Scenario: Streamed answer with usage
- **WHEN** a prompt is sent in streaming mode with `provider = "deepseek"` and a valid key
- **THEN** pieces of text are delivered progressively and the final response reports the number of input and output tokens

#### Scenario: Tool call on DeepSeek
- **WHEN** a conversation asking "What is the weather in La Paz?" is sent with a neutral definition of a tool `get_weather` whose parameters require a string `city`, with `provider = "deepseek"`
- **THEN** the response contains a tool call named `get_weather` whose arguments are a JSON object with `city` set, and the stop reason says the model wants to call tools

#### Scenario: DeepSeek overloaded, then success
- **WHEN** `provider = "deepseek"` is selected, the first request returns HTTP 503 and the second succeeds
- **THEN** the user sees one retry notice mentioning the server error and the answer is returned normally
