# Spec Delta

## MODIFIED Requirements

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

## ADDED Requirements

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
