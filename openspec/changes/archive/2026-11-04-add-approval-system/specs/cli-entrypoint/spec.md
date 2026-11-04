# Spec Delta

## ADDED Requirements

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
