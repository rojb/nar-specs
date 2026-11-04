# Spec Delta

## Purpose

Defines the built-in `glob` tool that lets the agent find files in the user's project by path pattern, which files it skips, and how its results are capped.

## ADDED Requirements

### Requirement: Find files by pattern
`glob` SHALL accept a required `pattern` and an optional `path` (the folder to search, absolute or relative to the working directory; default the working directory) and return the paths of the files under that folder whose path relative to it matches the pattern, one per line, sorted alphabetically. Returned paths SHALL be relative to the working directory when the file is inside it. It SHALL NOT require user approval.

#### Scenario: Recursive pattern
- **WHEN** the project has `main.py`, `src/app.py`, `src/utils/helpers.py` and `README.md`, and `glob` is called with `pattern` `**/*.py`
- **THEN** the result is a success listing `main.py`, `src/app.py` and `src/utils/helpers.py`, and not `README.md`

#### Scenario: Search inside a folder
- **WHEN** `glob` is called with `pattern` `*.py` and `path` `src`
- **THEN** only Python files under `src` are listed, shown with paths such as `src/app.py`

### Requirement: Pattern rules
In a pattern, `*` SHALL match any characters except `/`, `?` SHALL match one character except `/`, `[...]` SHALL match one character from the set, and a `**` path segment SHALL match zero or more folders. A pattern without `/` SHALL match file names at any depth.

#### Scenario: Single star stays in one folder
- **WHEN** `glob` is called with `pattern` `src/*.py` on a project with `src/app.py` and `src/utils/helpers.py`
- **THEN** only `src/app.py` is listed

#### Scenario: Pattern without a slash
- **WHEN** `glob` is called with `pattern` `test_*.py` on a project with `test_a.py` and `tests/unit/test_b.py`
- **THEN** both files are listed

### Requirement: Ignored files
`glob` SHALL skip files that git ignores when the searched folder is inside a git repository, and SHALL always skip the contents of the folders `.git`, `node_modules`, `.venv`, `__pycache__`, `.pytest_cache`, `.ruff_cache` and `.mypy_cache`.

#### Scenario: Gitignored file skipped
- **WHEN** the project is a git repository whose `.gitignore` contains `build/`, and `build/out.py` and `src/app.py` exist
- **THEN** `glob` with `pattern` `**/*.py` lists `src/app.py` and not `build/out.py`

#### Scenario: Noise folders skipped outside git
- **WHEN** the project is not a git repository and has `src/app.py` and `.venv/lib/site.py`
- **THEN** `glob` with `pattern` `**/*.py` lists `src/app.py` and not `.venv/lib/site.py`

### Requirement: Result cap and empty results
`glob` SHALL return at most 200 paths; when more files match, the result SHALL say how many matched in total and suggest a narrower pattern. When nothing matches, the result SHALL be a success saying no files matched the pattern.

#### Scenario: Too many matches
- **WHEN** 250 files match the pattern
- **THEN** the result lists 200 paths and states that 250 files matched

#### Scenario: No matches
- **WHEN** no file matches `**/*.rs`
- **THEN** the result is a success saying that no files match `**/*.rs`

### Requirement: Glob errors
`glob` SHALL return an error result, never crash, when the folder to search does not exist or is not a folder.

#### Scenario: Missing folder
- **WHEN** `glob` is called with `path` set to a folder that does not exist
- **THEN** the result is an error saying the folder was not found, including the resolved path
