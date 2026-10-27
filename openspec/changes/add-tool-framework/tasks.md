# Tasks

> Review workload: roughly 550 lines of code plus 500 lines of tests (about 1,000 changed lines), above the ~400-line guide. Suggested split into two PRs on the same branch family: PR A = groups 1-4 (dependencies, `ToolError`, contract, validation, truncation, registry); PR B = groups 5-8 (`read_file`, renderer, built-in registration, runner script, README). Group 9 (delivery) applies to each PR; group 10 runs once after both are merged.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-62 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-tool-framework/` on GitHub

## [nar]

## 1. Branch and dependencies

- [ ] 1.1 In `nar/`, update `main` and create a branch from Linear's RRG-62 branch name; verify `git status` shows the new branch
- [ ] 1.2 Add `rich` and `jsonschema` (lower bounds only) to `pyproject.toml` dependencies. RRG-61 also edits this list (`anthropic`, `openai`), so if the merge conflicts, resolve it by keeping both sets; verify `pip install -e ".[dev]"` succeeds and `python -c "import rich, jsonschema"` works

## 2. Errors and tool contract

- [ ] 2.1 Add `ToolError(NarError)` to `nar/errors.py` with a test in `tests/test_errors.py`; verify `pytest tests/test_errors.py` passes and `ToolError` is an instance of `NarError`
- [ ] 2.2 Implement `nar/tools/base.py` (`Tool` ABC with `name`, `description`, `parameters`, `requires_approval = False`, abstract `execute()`, `definition()` returning a deep copy; frozen `ToolContext(cwd)`; frozen `ToolResult` with `ok()`/`error()`; `ToolDefinition` TypedDict; `NAME_PATTERN`) from design.md §2-3 and a minimal `nar/tools/__init__.py`, with `tests/test_tools_base.py`; verify `pytest tests/test_tools_base.py` passes: a subclass without `execute` cannot be instantiated, `definition()` has exactly `name`, `description`, `parameters` and mutating it does not change the tool, `ToolResult.error("x").output == "x"` with `is_error` true

## 3. Validation and truncation

- [ ] 3.1 Implement `nar/tools/validation.py` (`validate_arguments(schema, arguments) -> list[str]` using `Draft202012Validator`, message table from design.md §4, no type coercion) with `tests/test_tools_validation.py`; verify `pytest tests/test_tools_validation.py` passes for: missing `path` gives `missing required argument 'path'`, `offset="ten"` names `offset` and `integer`, unknown argument, `minimum` violation, a valid call returns `[]`
- [ ] 3.2 Implement `nar/tools/limits.py` (`MAX_OUTPUT_CHARS = 30_000`, `truncate_output()` where head plus marker never exceeds the limit) with `tests/test_tools_limits.py`; verify `pytest tests/test_tools_limits.py` passes: 50,000 characters become `<= 30000` characters ending in `[output truncated: N characters omitted]` with `N >= 20000`, and 100 characters are returned unchanged

## 4. Registry

- [ ] 4.1 Implement `nar/tools/registry.py` (`ToolRegistry` with `register`, `get`, `names`, `definitions`, `execute`; name pattern and `check_schema` plus `type == "object"` checks at registration; `execute` pipeline lookup -> validate -> run in `try/except` -> truncate, per design.md §5) and re-export the public names from `nar/tools/__init__.py`, with `tests/test_tools_registry.py` using stub tools; verify `pytest tests/test_tools_registry.py` passes for: name `read-file!` is rejected naming the tool, a duplicate is rejected and the original tool is kept, an unknown tool `delete_everything` gives an error result listing the available names, export has exactly `name`/`description`/`parameters` in registration order, invalid arguments do not run the stub tool, a stub that raises `RuntimeError("boom")` gives an error result containing `RuntimeError` and `boom`, a stub that raises `ToolError` gives its bare message, and a stub returning 50,000 characters comes back truncated

## 5. read_file tool

- [ ] 5.1 Implement the reading path of `nar/tools/read_file.py` (`ReadFileTool` with schema `path`/`offset`/`limit`, `additionalProperties: false`, `requires_approval = False`; relative path resolved against `context.cwd`; one-pass reader with `utf-8` and `errors="replace"`; `{lineno}\t{text}` formatting; `[Showing lines X-Y of N]` header placed first; `(file is empty)`; `summary = "read N lines"`) with `tests/test_tools_read_file.py` using `tmp_path`; verify `pytest tests/test_tools_read_file.py` passes for: a 3-line file returns lines numbered 1, 2, 3, a relative path reads `<cwd>/src/app.py` through `ToolContext(cwd=tmp_path)`, a 100-line file with `offset=10, limit=5` returns lines 10-14 and states `lines 10-14 of 100`, a 5,000-line file without `limit` returns 2,000 lines and states `lines 1-2000 of 5000`, a latin-1 file is returned with replacement characters, and `ReadFileTool()` exposes `name == "read_file"`, a non-empty description, a parameters schema of type `object` and `requires_approval is False`
- [ ] 5.2 Add the error paths to `nar/tools/read_file.py` (not found with resolved path, directory, size above `MAX_FILE_BYTES`, NUL byte in the first 8 KiB, `offset` past the end, `OSError` message) and extend `tests/test_tools_read_file.py`; verify `pytest tests/test_tools_read_file.py` passes for: a missing path gives `file not found` with the resolved path, a directory gives `path is a directory`, a real PNG header gives `appears to be binary`, a file just over 10 MB (created sparse or with repeated bytes in `tmp_path`) gives `too large` and suggests a narrower tool such as a search, and `offset=150` on a 100-line file gives an error saying the file has only 100 lines

## 6. Terminal rendering

- [ ] 6.1 Implement `nar/ui/tool_view.py` (`ToolRenderer(console=None)` with `show_call` and `show_result`, `summarize_arguments(arguments, max_len=100)`, Unicode vs ASCII indicators chosen from `console.encoding`, `rich.markup.escape` on all dynamic text) and `nar/ui/__init__.py`, with `tests/test_tool_view.py` using `Console(file=io.StringIO(), width=100, force_terminal=False)`; verify `pytest tests/test_tool_view.py` passes for: `show_call("read_file", {"path": "pyproject.toml", "limit": 5})` prints a line containing `read_file` and `pyproject.toml`, a success result prints an indicator plus the summary (`read 5 lines`), an error result prints an error indicator plus the full message, a message containing `[bold]` is printed literally, a 500-character value is shortened with `...`, and a console with `encoding="cp1252"` prints the ASCII indicators without raising

## 7. Built-in registration, runner script and docs

- [ ] 7.1 Implement `nar/tools/builtin.py` (`register_builtin_tools(registry)` with the explicit list `ReadFileTool()`, `create_default_registry(cwd=None)`) and export `create_default_registry` from `nar/tools/__init__.py`, with tests in `tests/test_tools_registry.py`; verify `pytest tests/test_tools_registry.py` passes: the default registry's `names()` is `["read_file"]` and `definitions()` returns one object whose `name` is `read_file`, whose `description` equals the tool description and whose `parameters` equals its JSON Schema, and running `read_file` through the default registry without `path` returns an error result saying `path` is required without reading anything, and with `offset="ten"` returns an error naming `offset` and `integer`
- [ ] 7.2 Implement `scripts/run_tool.py` (Click; `tool_name` and JSON arguments defaulting to `{}`; invalid JSON or non-object prints `error: arguments are not valid JSON: <detail>` to stderr and exits 2; otherwise `show_call`, `registry.execute`, `show_result`, prints `result.output`; exit 1 on an error result) with `tests/test_run_tool_script.py` using `CliRunner`; verify `pytest tests/test_run_tool_script.py` passes: `read_file '{"path": "<tmp file>", "limit": 5}'` shows the call line and the first five numbered lines and exits 0, `'{not json'` exits non-zero with the JSON error, an unknown tool exits 1 listing the available tools
- [ ] 7.3 Update `nar/README.md` with one paragraph on `scripts/run_tool.py` (usage, argument JSON, exit codes); verify the documented example `python scripts/run_tool.py read_file '{"path": "pyproject.toml", "limit": 5}'` runs as written from the repo root

## 8. Cross-check

- [ ] 8.1 Run `pytest -q` with the tool test files and confirm in the output that no module under `nar/tools/` or `nar/ui/` imports `nar.providers` (`python -c "import nar.tools, nar.ui, sys; assert not any(m.startswith('nar.providers') for m in sys.modules)"` exits 0), and that `nar/cli/main.py` is unchanged (`git diff --stat main -- src/nar/cli src/nar/config src/nar/context` is empty)

## 9. Delivery

- [ ] 9.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR referencing RRG-62 (open a second PR for the second half if the review workload split is used)

## 10. Manual verification (RRG-62 "done when")

- [ ] 10.1 Walk through RRG-62's "done when" in a fresh terminal at the `nar/` repo root after `pip install -e ".[dev]"`, and record what was observed for each step. RRG-62's "done when" is: `read_file` runs from the registry and its schema is exported correctly. On Windows run the commands in Git Bash or PowerShell 7.3+ (Windows PowerShell 5.1 strips the inner double quotes from the JSON argument):
  - Schema export: `python -c "import json; from nar.tools import create_default_registry; print(json.dumps(create_default_registry().definitions(), indent=2))"` -> a list with one object holding exactly `name` = `read_file`, a non-empty `description`, and `parameters` of type `object` with `path` required and `offset`/`limit` integers
  - `python scripts/run_tool.py read_file '{"path": "pyproject.toml", "limit": 5}'` -> a call line with `read_file` and `pyproject.toml`, a success indicator with `read 5 lines`, then the header `[Showing lines 1-5 of N]` followed by five numbered lines; exit code 0
  - `python scripts/run_tool.py read_file '{"path": "pyproject.toml", "offset": 3, "limit": 2}'` -> lines numbered 3 and 4 and `lines 3-4 of N`
  - Relative path from a subfolder: `cd src && python ../scripts/run_tool.py read_file '{"path": "nar/errors.py"}'` -> reads `src/nar/errors.py`
  - Long file default limit: create a 5,000-line file, `python scripts/run_tool.py read_file '{"path": "big.txt"}'` -> 2,000 lines and `lines 1-2000 of 5000`
  - Truncation: read a file whose first 2,000 lines exceed 30,000 characters (for example 2,000 lines of 40 characters) -> output ends with `[output truncated: N characters omitted]` and the `[Showing lines ...]` header is still first
  - Unknown tool: `python scripts/run_tool.py delete_everything '{}'` -> red error indicator, message says the tool does not exist and lists `read_file`; exit 1; the script does not crash
  - Missing argument: `python scripts/run_tool.py read_file '{}'` -> error saying `path` is required; exit 1
  - Wrong type: `python scripts/run_tool.py read_file '{"path": "pyproject.toml", "offset": "ten"}'` -> error naming `offset` and `integer`; exit 1
  - Invalid JSON: `python scripts/run_tool.py read_file '{not json'` -> `error: arguments are not valid JSON: ...` on stderr; exit 2
  - Missing file: `python scripts/run_tool.py read_file '{"path": "nope.txt"}'` -> `file not found` with the resolved absolute path; exit 1
  - Directory: `python scripts/run_tool.py read_file '{"path": "src"}'` -> `path is a directory`; exit 1
  - Binary file: `python scripts/run_tool.py read_file '{"path": "<any .png>"}'` -> `file appears to be binary`; exit 1
  - Offset past end: `python scripts/run_tool.py read_file '{"path": "pyproject.toml", "offset": 9999}'` -> error saying the file has only N lines; exit 1
  - Windows redirect check (PowerShell): `python scripts/run_tool.py read_file '{"path": "pyproject.toml", "limit": 5}' > out.txt` -> no `UnicodeEncodeError`; `out.txt` shows the indicators
  - `nar --version` and `nar "hello"` still behave exactly as before this change (the CLI is untouched)
