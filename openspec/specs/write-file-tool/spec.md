# write-file-tool Specification

## Purpose
Defines the built-in `write_file` tool that lets the agent create a file or overwrite an existing one, including parent folder creation, the report of what it did and the errors it returns.

## Requirements

### Requirement: Write a file
`write_file` SHALL accept a required `path` (absolute, or relative to the working directory where `nar` was started) and a required `content` string, and SHALL save the content to that path exactly as given, encoded as UTF-8, with no newline conversion. If the file does not exist it SHALL be created; if it exists its previous content SHALL be replaced. The tool SHALL require user approval before running.

#### Scenario: Create a new file
- **WHEN** `write_file` is called with `path` set to a file that does not exist and `content` set to `"hello\n"`
- **THEN** the file is created containing exactly `hello` followed by a line break, and the result is a success

#### Scenario: Overwrite an existing file
- **WHEN** `write_file` is called with `path` set to an existing file and different `content`
- **THEN** the file contains only the new content and the result is a success

#### Scenario: Relative path
- **WHEN** `nar` was started in `/project` and `write_file` is called with `path` set to `src/app.py`
- **THEN** the file `/project/src/app.py` is written

#### Scenario: Empty content
- **WHEN** `write_file` is called with `content` set to an empty string
- **THEN** an empty file exists at the path and the result is a success

#### Scenario: Requires approval
- **WHEN** the built-in `write_file` tool is inspected
- **THEN** its `requires_approval` is set to true

### Requirement: Parent folders are created
`write_file` SHALL create any missing parent folders of `path` before writing the file.

#### Scenario: Missing parent folders
- **WHEN** `write_file` is called with `path` set to `docs/guides/intro.md` and neither `docs` nor `docs/guides` exists
- **THEN** both folders and the file are created and the result is a success

### Requirement: Write result report
A successful `write_file` result SHALL state whether the file was created or overwritten, the resolved path, and the size of the written content as a number of lines and a number of bytes. The terminal summary SHALL be a short one-line version of the same information.

#### Scenario: Created report
- **WHEN** `write_file` creates a new 3-line file of 24 bytes
- **THEN** the result says the file was created, includes the resolved path, and states 3 lines and 24 bytes

#### Scenario: Overwritten report
- **WHEN** `write_file` replaces an existing file
- **THEN** the result says the file was overwritten instead of created

#### Scenario: Terminal summary
- **WHEN** `write_file` succeeds on `notes.txt`
- **THEN** the terminal shows a line with `write_file` and `notes.txt`, followed by a success indicator and a short summary such as `created notes.txt (3 lines)`

### Requirement: Write errors
`write_file` SHALL return an error result, never crash, when the content cannot be written, and SHALL leave the filesystem unchanged when the error is detected before writing.

#### Scenario: Path is a directory
- **WHEN** `write_file` is called with the path of an existing directory
- **THEN** the result is an error saying the path is a directory, including the resolved path

#### Scenario: Parent path is a file
- **WHEN** `write_file` is called with `path` set to `app.py/x.txt` and `app.py` is an existing file
- **THEN** the result is an error saying a parent of the path is not a folder, and no folder or file is created

#### Scenario: Operating system refuses
- **WHEN** the operating system refuses the write, for example because the file is read-only
- **THEN** the result is an error naming the resolved path and the reason given by the operating system

### Requirement: Write tool registration
`write_file` SHALL be registered as a built-in tool so that it appears in the provider-neutral export and can be run by the developer tool runner.

#### Scenario: Listed in the export
- **WHEN** the registry of built-in tools is exported
- **THEN** the export contains an object named `write_file` with a description and a parameters schema that requires `path` and `content`
