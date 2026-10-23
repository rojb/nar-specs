# Spec Delta

## Purpose

Defines how `nar` talks to language models: one provider contract with a single internal message format, streamed and complete responses, tool definitions and tool calls translated for each supported provider, credentials, retries and error reporting.

## ADDED Requirements

### Requirement: Supported providers
`nar` SHALL support three LLM providers, selected by name: `anthropic` (Anthropic API), `openrouter` (OpenRouter API) and `ollama` (a local Ollama server). The same conversation SHALL be sendable to any of them without changes outside the provider and model selection.

#### Scenario: Same prompt on every provider
- **WHEN** the same prompt is sent once with `provider = "anthropic"`, once with `provider = "openrouter"` and once with `provider = "ollama"`, each with a valid model
- **THEN** each provider returns a text answer and nothing else in the request or the code had to change

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
API keys SHALL be read only from environment variables: `ANTHROPIC_API_KEY` for `anthropic` and `OPENROUTER_API_KEY` for `openrouter`. `ollama` SHALL need no key and SHALL use the server at `OLLAMA_HOST` when set, or `http://localhost:11434` otherwise. A missing key SHALL be reported before any request is sent.

#### Scenario: Missing API key
- **WHEN** `provider = "openrouter"` is selected and `OPENROUTER_API_KEY` is not set
- **THEN** `nar` exits with code 1 and a one-line error naming `OPENROUTER_API_KEY`, without contacting OpenRouter

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
Errors that retrying cannot fix SHALL fail at once with a one-line message: rejected credentials SHALL name the environment variable to check, an unknown model SHALL name the model and provider, and an unreachable Ollama server SHALL name the address and suggest starting Ollama.

#### Scenario: Invalid API key
- **WHEN** `ANTHROPIC_API_KEY` is set to a key the API rejects
- **THEN** `nar` makes no retry, exits with code 1 and the error names `ANTHROPIC_API_KEY`

#### Scenario: Unknown model
- **WHEN** `provider = "ollama"` and `model = "no-such-model"` are selected and the server does not have that model
- **THEN** `nar` exits with code 1 and the error names `no-such-model` and `ollama`

#### Scenario: Ollama not running
- **WHEN** `provider = "ollama"` is selected and nothing listens at the Ollama address
- **THEN** after the retries `nar` exits with code 1 and the error names the address and suggests starting Ollama

### Requirement: Developer provider runner
The `nar` repository SHALL include a developer script that sends a prompt to a chosen provider and model, streamed or not, optionally offering a sample tool, and prints the text and the internal response (tool calls, stop reason, token usage), so providers can be checked without the agent loop.

#### Scenario: Non-streamed run with a tool
- **WHEN** a developer runs the script with provider `ollama`, a local model that supports tools, the sample tool enabled, streaming off and the prompt "What is the weather in La Paz?"
- **THEN** the script prints a tool call to the sample tool with its arguments and the stop reason
