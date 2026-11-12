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
The `nar` command SHALL print usage information describing its modes and options when invoked with `--help`. The usage text, the option descriptions and Click's own words SHALL be shown in the interface language, worked out before the configuration is loaded.

#### Scenario: Help flag
- **WHEN** the user runs `nar --help`
- **THEN** the output describes how to start interactive mode and single mode, lists the available options, and the command exits with code 0

#### Scenario: Help in Spanish by default
- **WHEN** the user runs `nar --help` with no configuration
- **THEN** the usage text and the option descriptions are in Spanish

#### Scenario: Help in English
- **WHEN** the user runs `nar --language en --help`
- **THEN** the usage text and the option descriptions are in English

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

### Requirement: User-facing error reporting
Expected failures (such as invalid configuration) SHALL be reported as a short, human-readable message on standard error and a non-zero exit code, without a Python traceback. Unexpected internal failures SHALL also exit with a non-zero code.

#### Scenario: Expected failure
- **WHEN** `nar` is started and the configuration is invalid
- **THEN** a one-line error message explaining the problem is printed to standard error, no traceback is shown, and the command exits with code 1

#### Scenario: Debug details on request
- **WHEN** the user sets the environment variable `NAR_DEBUG=1` and an error occurs
- **THEN** the full traceback is printed in addition to the error message

### Requirement: Provider and model options
The `nar` command SHALL accept `--provider <name>` and `--model <name>` options in both modes, choosing the provider and model for that run; `--help` SHALL list them together with the allowed provider names (`anthropic`, `openrouter`, `ollama`, `deepseek`).

#### Scenario: Choose provider and model from the command line
- **WHEN** the user runs `nar --provider ollama --model qwen2.5-coder "hello"`
- **THEN** the prompt is answered by the `qwen2.5-coder` model of the local Ollama server, whatever the configuration files say

#### Scenario: Choose DeepSeek from the command line
- **WHEN** the user runs `nar --provider deepseek "hello"` with `DEEPSEEK_API_KEY` set
- **THEN** the prompt is answered by the documented default model of DeepSeek, whatever the configuration files say

#### Scenario: Help lists DeepSeek
- **WHEN** the user runs `nar --help`
- **THEN** the `--provider` option lists `deepseek` among the allowed names

#### Scenario: Invalid provider option
- **WHEN** the user runs `nar --provider openai "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `anthropic`, `openrouter`, `ollama` and `deepseek`

### Requirement: Approval option
The `nar` command SHALL accept an `--approval <level>` option in both modes, choosing the autonomy level (`ask`, `auto-edit` or `yolo`) for that run; `--help` SHALL list it together with the allowed levels. Questions asked because of the chosen level SHALL be shown and answered in the terminal in both modes.

#### Scenario: Choose the level from the command line
- **WHEN** the user runs `nar --approval auto-edit "add a docstring to src/app.py"` and the model calls `edit_file` on a file in the working directory
- **THEN** the file is edited without a question, whatever the configuration files say

#### Scenario: Invalid level
- **WHEN** the user runs `nar --approval always "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `ask`, `auto-edit` and `yolo`

#### Scenario: Help lists the option
- **WHEN** the user runs `nar --help`
- **THEN** the output lists `--approval` with the three allowed levels

#### Scenario: Question in single mode
- **WHEN** the user runs `nar "create hello.py"` in a terminal at the `ask` level and the model calls `write_file`
- **THEN** the user is asked in the terminal, and after the answer the request continues and the command exits with code 0

#### Scenario: Question in interactive mode
- **WHEN** the user is in interactive mode at the `ask` level and the model calls `shell`
- **THEN** the user is asked in the terminal, and after the answer the request continues and the prompt returns when it ends

### Requirement: Slash commands in interactive mode
In interactive mode, a line that starts with `/` followed by a command name (a letter, then letters, digits or `-`) and then either nothing or whitespace SHALL be handled as a command by `nar` and SHALL NOT be sent to the model or added to the conversation. Text after the name is the command's argument. A line that starts with `/` but is not of that form (for example a file path such as `/etc/hosts`) SHALL be treated as an ordinary request. A command name `nar` does not know SHALL print an error that lists the available commands and the session SHALL continue. Commands SHALL NOT be available in single mode, where the prompt is always a request. The available commands in this change are `/mcp`; further commands are added by later changes.

#### Scenario: Known command
- **WHEN** the user types `/mcp` at the `nar>` prompt
- **THEN** `nar` prints the MCP servers and returns to the prompt without calling the model

#### Scenario: Unknown command
- **WHEN** the user types `/foo` at the prompt
- **THEN** `nar` prints an error that `/foo` is not a command and lists the available commands, does not call the model, and shows the prompt again

#### Scenario: Path is not a command
- **WHEN** the user types `/etc/hosts what is this file?`
- **THEN** the line is sent to the model as a request

#### Scenario: Single mode
- **WHEN** the user runs `nar "/mcp"`
- **THEN** the text `/mcp` is sent to the model as the prompt

### Requirement: Language option
The `nar` command SHALL accept a `--language <code>` option in both modes, choosing the language the model replies in (`es` or `en`) for that run; `--help` SHALL list it together with the allowed values.

#### Scenario: Default reply language
- **WHEN** the user runs `nar "what does src/app.py do?"` with no configuration and no option
- **THEN** the model is told to reply in Spanish, and the answer is in Spanish even though the question is in English

#### Scenario: English from the command line
- **WHEN** the user runs `nar --language en "que hace src/app.py?"`
- **THEN** the model is told to reply in English, whatever the configuration files say, and the answer is in English

#### Scenario: Invalid language
- **WHEN** the user runs `nar --language fr "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `es` and `en`

#### Scenario: Help lists the option
- **WHEN** the user runs `nar --help`
- **THEN** the output lists `--language` with the two allowed values
