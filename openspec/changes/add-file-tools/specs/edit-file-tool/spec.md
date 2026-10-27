# Spec Delta

## Purpose

Defines the built-in `edit_file` tool that lets the agent change an existing file by replacing one exact piece of text, including the unique-match rule, the reported diff and the errors it returns.

## ADDED Requirements

### Requirement: Edit a file by exact replacement
`edit_file` SHALL accept a required `path` (absolute, or relative to the working directory where `nar` was started), a required `old_string` and a required `new_string`. It SHALL replace the single occurrence of `old_string` in the file with `new_string`. Matching SHALL be exact for case, whitespace and indentation. Line breaks SHALL match regardless of style: a line break in `old_string` matches either LF or CRLF in the file, because `read_file` shows every line break as LF. The replacement SHALL be written with the line-break style the file already uses. All other content of the file SHALL stay byte-for-byte identical, including its line endings. The tool SHALL require user approval before running.

#### Scenario: Replace a unique match
- **WHEN** `edit_file` is called on a file containing `timeout = 10` once, with `old_string` `timeout = 10` and `new_string` `timeout = 30`
- **THEN** the file contains `timeout = 30` in that place, everything else is unchanged, and the result is a success

#### Scenario: Relative path
- **WHEN** `nar` was started in `/project` and `edit_file` is called with `path` set to `src/app.py`
- **THEN** the file `/project/src/app.py` is edited

#### Scenario: Multi-line match
- **WHEN** `old_string` spans two lines and appears exactly once in the file
- **THEN** both lines are replaced by `new_string`

#### Scenario: Exact matching
- **WHEN** `old_string` is `Timeout = 10` and the file only contains `timeout = 10`
- **THEN** the result is an error saying `old_string` was not found and the file is unchanged

#### Scenario: Line endings preserved
- **WHEN** `edit_file` edits one line of a file that uses CRLF line endings and `old_string` contains no line break
- **THEN** every line ending in the file is still CRLF

#### Scenario: Multi-line match in a CRLF file
- **WHEN** a file uses CRLF line endings and `edit_file` is called with an `old_string` that spans two lines joined by LF and appears once in the file
- **THEN** the two lines are replaced, `new_string` is written with CRLF line endings, and every other line ending in the file is still CRLF

#### Scenario: Deleting text
- **WHEN** `new_string` is an empty string and `old_string` appears once
- **THEN** the matched text is removed from the file and the result is a success

#### Scenario: Requires approval
- **WHEN** the built-in `edit_file` tool is inspected
- **THEN** its `requires_approval` is set to true

### Requirement: Unique match rule
`edit_file` SHALL change the file only when `old_string` appears exactly once. When it appears zero times, the result SHALL be an error saying it was not found. When it appears more than once, the result SHALL be an error stating how many times it appears and asking for more surrounding context to make it unique. In both cases the file SHALL stay unchanged.

#### Scenario: Not found
- **WHEN** `old_string` does not appear in the file
- **THEN** the result is an error saying `old_string` was not found in the file, and the file is unchanged

#### Scenario: Several matches
- **WHEN** `old_string` appears 3 times in the file
- **THEN** the result is an error saying it was found 3 times and that more context is needed to make it unique, and the file is unchanged

### Requirement: Invalid edit requests
`edit_file` SHALL reject requests that cannot be a meaningful edit with an error result, leaving the file unchanged.

#### Scenario: Empty old string
- **WHEN** `edit_file` is called with `old_string` set to an empty string
- **THEN** the result is an error saying `old_string` must not be empty and suggesting `write_file` to create or replace whole files

#### Scenario: No change requested
- **WHEN** `edit_file` is called with `old_string` equal to `new_string`
- **THEN** the result is an error saying `old_string` and `new_string` are identical, and the file is unchanged

### Requirement: Edit errors
`edit_file` SHALL return an error result, never crash, when the file cannot be edited as text, and SHALL NOT create files.

#### Scenario: File not found
- **WHEN** `edit_file` is called with a path that does not exist
- **THEN** the result is an error saying the file was not found, including the resolved path, and no file is created

#### Scenario: Path is a directory
- **WHEN** `edit_file` is called with the path of a directory
- **THEN** the result is an error saying the path is a directory

#### Scenario: Binary file
- **WHEN** `edit_file` is called on a binary file such as a PNG image
- **THEN** the result is an error saying the file appears to be binary, and the file is unchanged

#### Scenario: Non-UTF-8 text
- **WHEN** `edit_file` is called on a text file that is not valid UTF-8
- **THEN** the result is an error saying the file is not valid UTF-8, and the file is unchanged

#### Scenario: Operating system refuses
- **WHEN** the operating system refuses to read or write the file, for example because it is read-only
- **THEN** the result is an error naming the resolved path and the reason given by the operating system

### Requirement: Edit result diff
A successful `edit_file` result SHALL include a unified diff of the change, with the path in its header and a few unchanged lines of context around each change, so the model and the user can see exactly what changed. The terminal summary SHALL be a short one-line description of the edit.

#### Scenario: Diff of a one-line change
- **WHEN** `edit_file` replaces `timeout = 10` with `timeout = 30` on line 12 of `config.py`
- **THEN** the result contains a unified diff that names `config.py`, shows the line `timeout = 10` as removed and `timeout = 30` as added, and shows unchanged lines around them

#### Scenario: Terminal summary
- **WHEN** `edit_file` succeeds on `config.py` with one line removed and one line added
- **THEN** the terminal shows a line with `edit_file` and `config.py`, followed by a success indicator and a short summary such as `edited config.py (+1 -1)`

### Requirement: Edit tool registration
`edit_file` SHALL be registered as a built-in tool so that it appears in the provider-neutral export and can be run by the developer tool runner.

#### Scenario: Listed in the export
- **WHEN** the registry of built-in tools is exported
- **THEN** the export contains an object named `edit_file` with a description and a parameters schema that requires `path`, `old_string` and `new_string`
