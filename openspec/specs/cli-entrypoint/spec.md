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
Running `nar` with a prompt argument SHALL process that one prompt and exit without entering interactive mode.

#### Scenario: Single prompt
- **WHEN** the user runs `nar "explain this repository"`
- **THEN** the prompt is processed once, its result is printed, and the command exits with code 0 without waiting for further input

### Requirement: User-facing error reporting
Expected failures (such as invalid configuration) SHALL be reported as a short, human-readable message on standard error and a non-zero exit code, without a Python traceback. Unexpected internal failures SHALL also exit with a non-zero code.

#### Scenario: Expected failure
- **WHEN** `nar` is started and the configuration is invalid
- **THEN** a one-line error message explaining the problem is printed to standard error, no traceback is shown, and the command exits with code 1

#### Scenario: Debug details on request
- **WHEN** the user sets the environment variable `NAR_DEBUG=1` and an error occurs
- **THEN** the full traceback is printed in addition to the error message
