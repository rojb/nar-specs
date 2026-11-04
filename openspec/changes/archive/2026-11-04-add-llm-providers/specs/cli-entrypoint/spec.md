# Spec Delta

## MODIFIED Requirements

### Requirement: Single mode
Running `nar` with a prompt argument SHALL process that one prompt and exit without entering interactive mode. Until the agent loop exists, processing means sending the prompt to the selected provider and model and printing the answer to standard output as it is generated.

#### Scenario: Single prompt
- **WHEN** the user runs `nar "explain this repository"`
- **THEN** the prompt is processed once, its result is printed, and the command exits with code 0 without waiting for further input

#### Scenario: Answer is streamed
- **WHEN** the user runs `nar "count from 1 to 20"` with a reachable provider
- **THEN** the answer appears on standard output progressively while it is generated, ends with a newline, and the command exits with code 0

#### Scenario: Answer cut by the output limit
- **WHEN** the model stops because it reached the output token limit (for example, a reasoning model that spends the whole limit before writing text)
- **THEN** whatever text was generated is printed, a warning on standard error says the answer was cut off by the output token limit, and the command exits with code 0

#### Scenario: Provider failure in single mode
- **WHEN** the user runs `nar "hello"` and the provider cannot be used (for example, its API key is missing)
- **THEN** a one-line error is printed to standard error, nothing is printed to standard output, and the command exits with code 1

## ADDED Requirements

### Requirement: Provider and model options
The `nar` command SHALL accept `--provider <name>` and `--model <name>` options in both modes, choosing the provider and model for that run; `--help` SHALL list them together with the allowed provider names.

#### Scenario: Choose provider and model from the command line
- **WHEN** the user runs `nar --provider ollama --model qwen2.5-coder "hello"`
- **THEN** the prompt is answered by the `qwen2.5-coder` model of the local Ollama server, whatever the configuration files say

#### Scenario: Invalid provider option
- **WHEN** the user runs `nar --provider openai "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `anthropic`, `openrouter` and `ollama`
