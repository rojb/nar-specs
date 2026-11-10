# llm-providers Specification

## Purpose
Defines how `nar` talks to language models: one provider contract with a single internal message format, streamed and complete responses, tool definitions and tool calls translated for each supported provider, credentials, retries and error reporting.

## Requirements

### Requirement: Supported providers
`nar` SHALL support four LLM providers, selected by name: `anthropic` (Anthropic API), `openrouter` (OpenRouter API), `ollama` (a local Ollama server) and `deepseek` (DeepSeek API). The same conversation SHALL be sendable to any of them without changes outside the provider and model selection, including conversations in which the model calls tools over several turns.

#### Scenario: Same prompt on every provider
- **WHEN** the same prompt is sent once with `provider = "anthropic"`, once with `provider = "openrouter"`, once with `provider = "ollama"` and once with `provider = "deepseek"`, each with a valid model
- **THEN** each provider returns a text answer and nothing else in the request or the code had to change

#### Scenario: Multi-turn tool conversation on DeepSeek
- **WHEN** with `provider = "deepseek"` the model answers a prompt with a tool call, the tool result is sent back, and the model then calls a second tool and finally answers in text
- **THEN** every request in the conversation is accepted by the provider and the final answer refers to the tool results

### Requirement: Internal message format
Conversations SHALL be described with one provider-independent format: an optional system prompt plus an ordered list of user messages, assistant messages (text and zero or more tool calls, each with an id, a tool name and arguments as a JSON object) and tool-result messages (the id of the tool call they answer, the result text and whether it is an error). Responses SHALL use the same format.

#### Scenario: Tool round trip
- **WHEN** a conversation contains a user message, an assistant message with a tool call `read_file` with id `call_1`, and a tool-result message for `call_1`
- **THEN** every provider accepts the conversation and the model's next answer can refer to the tool result

#### Scenario: Several tool calls in one turn
- **WHEN** an assistant message carries two tool calls and the conversation then contains one tool-result message for each of them
- **THEN** every provider accepts the conversation

### Requirement: Complete responses
A provider SHALL be able to return a whole response in one piece. The response SHALL contain the assistant text, the tool calls requested by the model, the reason the model stopped (finished, wants to call tools, or reached the output limit) and, when the provider reports it, the number of input and output tokens used.

#### Scenario: Text answer
- **WHEN** a conversation with one user message "Say hello" is sent without tools
- **THEN** the response contains non-empty text, no tool calls, and the stop reason says the model finished

### Requirement: Streamed responses
A provider SHALL be able to stream a response: text SHALL be delivered in pieces as the model generates it, and the stream SHALL end with the same complete response a non-streamed call would return, including any tool calls assembled from the stream.

#### Scenario: Text arrives incrementally
- **WHEN** a long answer is requested in streaming mode
- **THEN** pieces of text are delivered before the model has finished, and their concatenation equals the text of the final complete response

#### Scenario: Tool calls in a stream
- **WHEN** the model requests a tool call while streaming
- **THEN** the final complete response contains that tool call with its id, name and fully assembled arguments

### Requirement: Tool definitions accepted in neutral form
Providers SHALL accept tool definitions in the provider-neutral form exported by the tool registry (objects with `name`, `description` and `parameters`, where `parameters` is a JSON Schema) and SHALL translate them to each provider's own format. Tool calls in the provider's answer SHALL be translated back to the internal format.

#### Scenario: Model asks for a tool
- **WHEN** a conversation asking "What is the weather in La Paz?" is sent together with a neutral definition of a tool `get_weather` whose parameters require a string `city`
- **THEN** with a model that supports tools, the response contains a tool call named `get_weather` whose arguments are a JSON object with `city` set, and the stop reason says the model wants to call tools

#### Scenario: Unparseable tool arguments
- **WHEN** the provider returns a tool call whose arguments are not valid JSON
- **THEN** the tool call is still returned with its id and name, with an empty arguments object, and `nar` does not crash

### Requirement: Credentials from the environment
API keys SHALL be read only from environment variables: `ANTHROPIC_API_KEY` for `anthropic`, `OPENROUTER_API_KEY` for `openrouter` and `DEEPSEEK_API_KEY` for `deepseek`. `ollama` SHALL need no key and SHALL use the server at `OLLAMA_HOST` when set, or `http://localhost:11434` otherwise. A missing key SHALL be reported before any request is sent, unless a terminal is attached and the user provides the key when asked.

#### Scenario: Missing API key
- **WHEN** `provider = "openrouter"` is selected, `OPENROUTER_API_KEY` is not set and no terminal is attached
- **THEN** `nar` exits with code 1 and a one-line error naming `OPENROUTER_API_KEY`, without contacting OpenRouter

#### Scenario: Missing DeepSeek API key
- **WHEN** `provider = "deepseek"` is selected, `DEEPSEEK_API_KEY` is not set or is empty, and no terminal is attached
- **THEN** `nar` exits with code 1 and a one-line error naming `DEEPSEEK_API_KEY`, without contacting DeepSeek

#### Scenario: Custom Ollama host
- **WHEN** `provider = "ollama"` is selected and `OLLAMA_HOST` is `http://192.168.1.20:11434`
- **THEN** requests go to that server

### Requirement: Retries with exponential backoff
A request that fails with a rate limit (HTTP 429), a server error (HTTP 5xx, including Anthropic's 529 overloaded), a connection error or a timeout SHALL be retried up to 3 times, waiting longer before each retry (exponential backoff). When the provider says how long to wait, that wait SHALL be used instead, up to 60 seconds. Each retry SHALL be announced to the user with the reason, the wait and the attempt number.

#### Scenario: Rate limited, then success
- **WHEN** the first request returns HTTP 429 and the second succeeds
- **THEN** the user sees one retry notice mentioning the rate limit, and the answer is returned normally

#### Scenario: Retries exhausted
- **WHEN** every attempt returns HTTP 503
- **THEN** after the initial attempt and 3 retries `nar` stops with a one-line error naming the provider and the last failure, and exits with code 1

#### Scenario: Stream interrupted after output started
- **WHEN** a streamed response fails after some text has already been delivered
- **THEN** the request is not retried and the failure is reported as an error, so the user never sees the same text twice

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

### Requirement: Developer provider runner
The `nar` repository SHALL include a developer script that sends a prompt to a chosen provider and model, streamed or not, optionally offering a sample tool, and prints the text and the internal response (tool calls, stop reason, token usage), so providers can be checked without the agent loop.

#### Scenario: Non-streamed run with a tool
- **WHEN** a developer runs the script with provider `ollama`, a local model that supports tools, the sample tool enabled, streaming off and the prompt "What is the weather in La Paz?"
- **THEN** the script prints a tool call to the sample tool with its arguments and the stop reason

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

### Requirement: A provider can check its access without a model call
Every provider SHALL be able to check, without sending a conversation or spending tokens, that its server answers, that its key is accepted and that its model exists. The check SHALL make a single attempt with a short timeout, and SHALL fail with the same one-line messages a request uses. When the model does not exist and the provider lists its models, the error SHALL suggest up to three close names. For Ollama, a model that is not installed SHALL suggest `ollama pull <model>`, and a name without a tag SHALL match the `latest` tag.

#### Scenario: A rejected key
- **WHEN** a provider is checked with a key its server rejects
- **THEN** the check fails with the message that names the key's environment variable

#### Scenario: A model that does not exist
- **WHEN** a provider is checked with a misspelled model name such as `deepseek-flsh`
- **THEN** the check fails with `model 'deepseek-flsh' was not found by deepseek` and suggests `deepseek-flash`

#### Scenario: Ollama is not running
- **WHEN** Ollama is checked and nothing listens at its host
- **THEN** the check fails once, without retries, with the message that says how to start it

#### Scenario: An Ollama model that is not installed
- **WHEN** Ollama is checked with a model it has not pulled
- **THEN** the check fails with a message that suggests `ollama pull <model>`

#### Scenario: Everything answers
- **WHEN** the server answers, the key is accepted and the model is listed
- **THEN** the check passes and no model call is made

### Requirement: A missing key is asked for in the terminal
When the selected provider's key variable is not set, and both standard input and standard error are terminals, `nar` SHALL ask for the key on standard error with hidden input, at startup and when `/model` switches to that provider. The key SHALL be checked with the provider's access check before it is used. When the check passes, `nar` SHALL ask whether to save it. On Windows, saving SHALL set a user environment variable with the variable's name. On other systems, `nar` SHALL NOT save it and SHALL show the `export` line to add, with a placeholder instead of the key. The key SHALL NOT be written to any file, session or message. An empty answer, end of input or Ctrl+C SHALL cancel and report the missing variable as before.

#### Scenario: Pasting a valid key and saving it
- **WHEN** `nar --provider deepseek` starts in a terminal without `DEEPSEEK_API_KEY`, the user pastes a key DeepSeek accepts and answers `y`
- **THEN** the session starts with that key, `DEEPSEEK_API_KEY` is saved for the Windows user, and the key does not appear in the output

#### Scenario: Using a key once
- **WHEN** the user pastes a key DeepSeek accepts and answers `n`
- **THEN** the session starts with that key, and nothing is saved

#### Scenario: A rejected key
- **WHEN** the user pastes a key the provider rejects
- **THEN** `nar` reports the rejected key in one line, does not save it, and does not start the session; in `/model`, the previous provider stays

#### Scenario: Cancelling
- **WHEN** the user presses Enter with no key, or Ctrl+C
- **THEN** `nar` reports the missing variable in one line, as without a terminal

#### Scenario: No terminal
- **WHEN** the key is missing and the input is piped
- **THEN** `nar` asks nothing and reports the missing variable, as before

#### Scenario: Switching with /model
- **WHEN** the user types `/model anthropic` without `ANTHROPIC_API_KEY` in a terminal, and pastes a key Anthropic accepts
- **THEN** the switch happens with that key, after the same save question
