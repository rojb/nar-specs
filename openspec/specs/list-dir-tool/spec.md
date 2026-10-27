# list-dir-tool Specification

## Purpose
Defines the built-in `list_dir` tool that lets the agent see the files and folders inside a folder of the user's project, skipping folders that are noise for a coding agent.

## Requirements

### Requirement: List a folder
`list_dir` SHALL accept an optional `path` (absolute, or relative to the working directory where `nar` was started; default: the working directory) and return the direct entries of that folder, not its descendants. Folder names SHALL end with `/` so they are distinguishable from files, and entries SHALL be sorted by name, ignoring case. Hidden entries (names starting with `.`) SHALL be listed unless they are ignored by the rule below. The tool SHALL NOT require user approval.

#### Scenario: List with default path
- **WHEN** `nar` was started in `/project` containing `README.md`, `src/` and `pyproject.toml`, and `list_dir` is called without `path`
- **THEN** the result is a success listing `pyproject.toml`, `README.md` and `src/` in that order, one entry per line

#### Scenario: Relative path
- **WHEN** `nar` was started in `/project` and `list_dir` is called with `path` set to `src`
- **THEN** the entries of `/project/src` are listed

#### Scenario: Folders are distinguished
- **WHEN** a folder contains a file `main.py` and a subfolder `utils`
- **THEN** the result lists `main.py` and `utils/`

#### Scenario: Not recursive
- **WHEN** `list_dir` is called on a folder whose subfolder `utils` contains `helpers.py`
- **THEN** the result lists `utils/` and does not list `helpers.py`

#### Scenario: Requires no approval
- **WHEN** the built-in `list_dir` tool is inspected
- **THEN** its `requires_approval` is set to false

### Requirement: Ignored folders
`list_dir` SHALL omit entries named `.git`, `node_modules`, `.venv`, `__pycache__`, `.pytest_cache`, `.ruff_cache` and `.mypy_cache` from the result. The rule applies to the entries being listed; a call whose `path` is itself one of those folders SHALL list its contents normally.

#### Scenario: Noise folders skipped
- **WHEN** a folder contains `.git/`, `node_modules/`, `.venv/`, `__pycache__/` and `src/`
- **THEN** the result lists only `src/`

#### Scenario: Explicit path inside an ignored folder
- **WHEN** `list_dir` is called with `path` set to `node_modules`
- **THEN** the entries of `node_modules` are listed

### Requirement: Empty folder result
When a folder has no entries to show, `list_dir` SHALL return a success result whose text says the folder is empty, rather than an empty output.

#### Scenario: Empty folder
- **WHEN** `list_dir` is called on a folder with no entries
- **THEN** the result is a success stating that the folder is empty

#### Scenario: Only ignored entries
- **WHEN** `list_dir` is called on a folder that contains only `__pycache__/`
- **THEN** the result is a success stating that the folder is empty

### Requirement: List errors
`list_dir` SHALL return an error result, never crash, when the folder cannot be listed.

#### Scenario: Folder not found
- **WHEN** `list_dir` is called with a path that does not exist
- **THEN** the result is an error saying the folder was not found, including the resolved path

#### Scenario: Path is a file
- **WHEN** `list_dir` is called with the path of a file
- **THEN** the result is an error saying the path is not a directory

#### Scenario: Operating system refuses
- **WHEN** the operating system refuses to read the folder, for example for lack of permission
- **THEN** the result is an error naming the resolved path and the reason given by the operating system

### Requirement: List tool registration
`list_dir` SHALL be registered as a built-in tool so that it appears in the provider-neutral export and can be run by the developer tool runner.

#### Scenario: Listed in the export
- **WHEN** the registry of built-in tools is exported
- **THEN** the export contains an object named `list_dir` with a description and a parameters schema that has no required arguments
