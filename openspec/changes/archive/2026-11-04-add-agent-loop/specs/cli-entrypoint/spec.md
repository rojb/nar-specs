# Spec Delta

## MODIFIED Requirements

### Requirement: Interactive mode
Running `nar` with no prompt argument SHALL start an interactive session that repeatedly reads user input until the user leaves it. Each non-empty line SHALL be handled as a request to the agent, and the conversation SHALL be kept for the whole session.

#### Scenario: Start and exit interactive mode
- **WHEN** the user runs `nar` with no arguments and then types `exit`
- **THEN** the session starts showing a prompt, ends after `exit`, and the command exits with code 0

#### Scenario: End of input
- **WHEN** the user is in interactive mode and sends end-of-input (Ctrl+D, or Ctrl+Z then Enter on Windows) or presses Ctrl+C at an empty prompt
- **THEN** the session ends without printing a traceback and the command exits with code 0

#### Scenario: Empty input is ignored
- **WHEN** the user submits an empty or whitespace-only line in interactive mode
- **THEN** nothing is processed and the prompt is shown again

#### Scenario: Ctrl+C during a request
- **WHEN** the user presses Ctrl+C while the agent is working on a request
- **THEN** that request is cancelled, the session continues, and the prompt is shown again

### Requirement: Single mode
Running `nar` with a prompt argument SHALL process that one prompt and exit without entering interactive mode. Processing means running the agent on the prompt with the selected provider and model and printing the answer to standard output as it is generated.

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

#### Scenario: Single mode uses tools
- **WHEN** the user runs `nar "read pyproject.toml and tell me the version"` in a project with a `pyproject.toml`
- **THEN** the agent reads the file with `read_file`, prints an answer that states the version, and the command exits with code 0

#### Scenario: Turn limit in single mode
- **WHEN** a single-mode request stops at the turn limit
- **THEN** the message about the limit is printed to standard error and the command exits with code 1

#### Scenario: Ctrl+C in single mode
- **WHEN** the user presses Ctrl+C while a single-mode request is running
- **THEN** the request is cancelled without a traceback and the command exits with code 130
