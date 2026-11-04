# cli-entrypoint Specification

## Purpose
Defines the `nar` terminal command: how users install and launch it, its interactive and single-prompt modes, and how failures are reported to them.

## Requirements

### Requirement: Installable `nar` command
The project SHALL be installable into a Python 3.11+ environment so that a `nar` command becomes available on the user's PATH and can be run from any directory.

#### Scenario: Run from an unrelated directory
- **WHEN** the package is installed in the active environment and the user runs `nar --version` from a directory outside the repository
- **THEN** the command prints the program name and its version and exits with code 0

### Requirement: Help output
The `nar` command SHALL print usage information describing its modes and options when invoked with `--help`.

#### Scenario: Help flag
- **WHEN** the user runs `nar --help`
- **THEN** the output describes how to start interactive mode and single mode, lists the available options, and the command exits with code 0

### Requirement: Interactive mode
Running `nar` with no prompt argument SHALL start an interactive session that repeatedly reads user input until the user leaves it.

#### Scenario: Start and exit interactive mode
- **WHEN** the user runs `nar` with no arguments and then types `exit`
- **THEN** the session starts showing a prompt, ends after `exit`, and the command exits with code 0

#### Scenario: End of input
- **WHEN** the user is in interactive mode and sends end-of-input (Ctrl+D, or Ctrl+Z then Enter on Windows) or presses Ctrl+C at an empty prompt
- **THEN** the session ends without printing a traceback and the command exits with code 0

#### Scenario: Empty input is ignored
- **WHEN** the user submits an empty or whitespace-only line in interactive mode
- **THEN** nothing is processed and the prompt is shown again

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

### Requirement: User-facing error reporting
Expected failures (such as invalid configuration) SHALL be reported as a short, human-readable message on standard error and a non-zero exit code, without a Python traceback. Unexpected internal failures SHALL also exit with a non-zero code.

#### Scenario: Expected failure
- **WHEN** `nar` is started and the configuration is invalid
- **THEN** a one-line error message explaining the problem is printed to standard error, no traceback is shown, and the command exits with code 1

#### Scenario: Debug details on request
- **WHEN** the user sets the environment variable `NAR_DEBUG=1` and an error occurs
- **THEN** the full traceback is printed in addition to the error message

### Requirement: Provider and model options
The `nar` command SHALL accept `--provider <name>` and `--model <name>` options in both modes, choosing the provider and model for that run; `--help` SHALL list them together with the allowed provider names.

#### Scenario: Choose provider and model from the command line
- **WHEN** the user runs `nar --provider ollama --model qwen2.5-coder "hello"`
- **THEN** the prompt is answered by the `qwen2.5-coder` model of the local Ollama server, whatever the configuration files say

#### Scenario: Invalid provider option
- **WHEN** the user runs `nar --provider openai "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `anthropic`, `openrouter` and `ollama`
