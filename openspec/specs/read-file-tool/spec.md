# read-file-tool Specification

## Purpose
Defines the built-in `read_file` tool that lets the agent read text files from the user's project, including line ranges, line numbering and the errors it reports.

## Requirements

### Requirement: Read a text file
`read_file` SHALL accept a required `path` (absolute, or relative to the working directory where `nar` was started) and return the file's text with each line prefixed by its 1-based line number. It SHALL NOT require user approval.

#### Scenario: Read whole file
- **WHEN** `read_file` is called with `path` set to a 3-line UTF-8 file
- **THEN** the result is a success containing the three lines, each prefixed with its line number (1, 2, 3)

#### Scenario: Relative path
- **WHEN** `nar` was started in `/project` and `read_file` is called with `path` set to `src/app.py`
- **THEN** the file `/project/src/app.py` is read

### Requirement: Line range
`read_file` SHALL accept an optional `offset` (1-based first line, default 1) and an optional `limit` (maximum number of lines, default 2000). When the range does not cover the whole file, the result SHALL say which lines were shown and how many lines the file has in total.

#### Scenario: Partial read
- **WHEN** `read_file` is called on a 100-line file with `offset` 10 and `limit` 5
- **THEN** the result contains lines 10 to 14 numbered 10 to 14, and states that lines 10-14 of 100 are shown

#### Scenario: Offset past the end
- **WHEN** `read_file` is called on a 100-line file with `offset` 150
- **THEN** the result is an error saying the file has only 100 lines

#### Scenario: Default limit on a long file
- **WHEN** `read_file` is called without `limit` on a 5,000-line file
- **THEN** the first 2,000 lines are returned and the result states that lines 1-2000 of 5000 are shown

### Requirement: Read errors
`read_file` SHALL return an error result, never crash, when the file cannot be read as text.

#### Scenario: File not found
- **WHEN** `read_file` is called with a path that does not exist
- **THEN** the result is an error saying the file was not found, including the resolved path

#### Scenario: Path is a directory
- **WHEN** `read_file` is called with the path of a directory
- **THEN** the result is an error saying the path is a directory

#### Scenario: Binary file
- **WHEN** `read_file` is called on a binary file such as a PNG image
- **THEN** the result is an error saying the file appears to be binary

#### Scenario: File too large
- **WHEN** `read_file` is called on a file larger than 10 MB
- **THEN** the result is an error saying the file is too large and suggesting a narrower tool such as a search

#### Scenario: Non-UTF-8 text
- **WHEN** `read_file` is called on a text file that is not valid UTF-8
- **THEN** the file is still returned, with undecodable bytes replaced, instead of failing
