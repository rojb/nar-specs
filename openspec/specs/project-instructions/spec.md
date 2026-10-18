# project-instructions Specification

## Purpose
Defines how `nar` finds a project's `AGENTS.md` file and makes its contents available as project-specific instructions for the agent.

## Requirements

### Requirement: AGENTS.md discovery
At startup, `nar` SHALL look for a file named `AGENTS.md` in the current working directory and, if it is not there, in each parent directory up to and including the root of the enclosing git repository. The first file found SHALL be used. If the current directory is not inside a git repository, only the current directory SHALL be checked.

#### Scenario: File in the working directory
- **WHEN** `nar` starts in a directory that contains `AGENTS.md`
- **THEN** that file's contents are loaded as the project instructions

#### Scenario: File at the repository root
- **WHEN** `nar` starts in a subdirectory of a git repository and only the repository root contains `AGENTS.md`
- **THEN** the repository root's `AGENTS.md` is loaded

#### Scenario: No file
- **WHEN** no `AGENTS.md` is found
- **THEN** `nar` starts normally with no project instructions and shows no error

### Requirement: Instructions available to the agent
The loaded project instructions SHALL be available, together with the path they came from, to the rest of the application so the agent can include them in its context.

#### Scenario: Startup notice
- **WHEN** an `AGENTS.md` file is loaded in interactive mode
- **THEN** the startup output tells the user which `AGENTS.md` file is in use

### Requirement: Oversized or unreadable file
`nar` SHALL NOT fail to start because of the instructions file. A file that cannot be read SHALL be skipped with a warning, and a file larger than 64 KB SHALL be truncated to that size with a warning.

#### Scenario: Unreadable file
- **WHEN** `AGENTS.md` exists but cannot be read or is not valid UTF-8
- **THEN** a warning naming the file is shown and `nar` continues without project instructions

#### Scenario: Very large file
- **WHEN** `AGENTS.md` is larger than 64 KB
- **THEN** only its first 64 KB are used and a warning says the file was truncated
