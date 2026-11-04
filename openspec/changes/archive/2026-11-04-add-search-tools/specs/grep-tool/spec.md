# Spec Delta

## Purpose

Defines the built-in `grep` tool that lets the agent search the contents of the user's project with a regular expression, how matches are reported, which files it skips, and how its results are capped.

## ADDED Requirements

### Requirement: Search file contents
`grep` SHALL accept a required `pattern` (a regular expression, Python syntax) and an optional `path` (a folder or a single file, absolute or relative to the working directory; default the working directory). It SHALL return every line that contains a match, one per line, as `path:line:text`, where `path` is relative to the working directory when the file is inside it and `line` is the 1-based line number. Files SHALL be searched in alphabetical path order. It SHALL NOT require user approval.

#### Scenario: Find a definition
- **WHEN** `src/nar/config/loader.py` contains the line `def load_config(cwd: Path) -> LoadedConfig:` at line 41 and `grep` is called with `pattern` `def load_config`
- **THEN** the result is a success containing `src/nar/config/loader.py:41:def load_config(cwd: Path) -> LoadedConfig:`

#### Scenario: Search a single file
- **WHEN** `grep` is called with `pattern` `import` and `path` set to one file
- **THEN** only matching lines of that file are returned

### Requirement: Search options
`grep` SHALL accept an optional `glob` that limits the search to files whose path matches it, with the same pattern rules as the `glob` tool, and an optional `ignore_case` (default false) that makes the match case-insensitive.

#### Scenario: Limit to Python files
- **WHEN** `TODO` appears in `app.py` and in `notes.md`, and `grep` is called with `pattern` `TODO` and `glob` `*.py`
- **THEN** only the match in `app.py` is returned

#### Scenario: Case-insensitive search
- **WHEN** a file contains `Timeout = 10` and `grep` is called with `pattern` `timeout` and `ignore_case` true
- **THEN** that line is returned

### Requirement: Invalid regular expression
An invalid `pattern` SHALL produce an error result that includes the regular expression's error, without searching any file.

#### Scenario: Unbalanced parenthesis
- **WHEN** `grep` is called with `pattern` `def (`
- **THEN** the result is an error saying the pattern is not a valid regular expression and why

### Requirement: Skipped files
`grep` SHALL skip the same files as `glob` (files ignored by git inside a repository, and the noise folders), files that look binary (a NUL byte in their first 8 KiB) and files larger than 10 MB. Text that is not valid UTF-8 SHALL still be searched, with invalid bytes replaced.

#### Scenario: Binary file skipped
- **WHEN** a PNG file and a text file both contain the bytes of `nar`, and `grep` is called with `pattern` `nar`
- **THEN** only the text file's line is returned

#### Scenario: Gitignored file skipped
- **WHEN** the project is a git repository that ignores `dist/`, and `dist/app.js` and `src/app.js` both contain `render`
- **THEN** `grep` with `pattern` `render` returns only the line from `src/app.js`

### Requirement: Result caps and empty results
`grep` SHALL return at most 100 matching lines; when there are more, the result SHALL say that it stopped at 100 and suggest narrowing the search. Each shown line SHALL be cut to 300 characters with a marker. When nothing matches, the result SHALL be a success saying no lines matched the pattern.

#### Scenario: Too many matches
- **WHEN** 150 lines match
- **THEN** the result contains 100 of them and says the output stopped at 100 matches

#### Scenario: Very long line
- **WHEN** a matching line has 5,000 characters
- **THEN** the shown text is cut to 300 characters followed by a marker that it was shortened

#### Scenario: No matches
- **WHEN** no line matches `class Foo`
- **THEN** the result is a success saying that no lines match `class Foo`

### Requirement: Grep errors
`grep` SHALL return an error result, never crash, when `path` does not exist.

#### Scenario: Missing path
- **WHEN** `grep` is called with `path` set to something that does not exist
- **THEN** the result is an error saying the path was not found, including the resolved path
