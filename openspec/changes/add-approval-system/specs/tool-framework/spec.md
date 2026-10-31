# Spec Delta

## ADDED Requirements

### Requirement: Tool call preview
Every tool SHALL be able to describe a specific call without carrying it out: a short title, optional detail text, an optional diff, and, when the tool already knows the call would be refused without effect, the error message it would return. The default description SHALL be the tool name and the call's arguments, so a tool that provides nothing is still described correctly. Producing a description SHALL NOT change files, run commands or contact services. When the description cannot be produced because of an unexpected failure, the default description SHALL be used.

#### Scenario: Default description
- **WHEN** a tool that provides no description of its own is described for a call with arguments `{"query": "nar"}`
- **THEN** the description names the tool and shows the argument `query` with the value `nar`

#### Scenario: Description changes nothing
- **WHEN** `edit_file` is described for a valid call
- **THEN** the file is byte-for-byte identical afterwards

#### Scenario: Known failure is reported in the description
- **WHEN** a tool is described for a call it would refuse without effect, such as `edit_file` with an `old_string` that is not in the file
- **THEN** the description carries the same error message the tool returns when run

#### Scenario: Description fails unexpectedly
- **WHEN** producing a tool's own description raises an unexpected exception
- **THEN** the default description is used and `nar` keeps running

### Requirement: Declaring calls that only edit project files
A tool SHALL be able to declare, for a specific call, that the call only changes files inside the working directory where `nar` was started (and not inside its `.git` or `.nar` folders). By default a tool SHALL NOT make this declaration. File paths SHALL be resolved (relative parts, `..` and symbolic links) before they are compared with the working directory.

#### Scenario: Default tools make no declaration
- **WHEN** a tool that does not declare anything is asked whether a call only edits project files
- **THEN** the answer is no

#### Scenario: Path escapes through parent references
- **WHEN** `write_file` is asked about a call whose path is `../outside.txt`
- **THEN** the answer is no

#### Scenario: Path inside the working directory
- **WHEN** `edit_file` is asked about a call whose path is `src/app.py` and `nar` was started in `/project`
- **THEN** the answer is yes
