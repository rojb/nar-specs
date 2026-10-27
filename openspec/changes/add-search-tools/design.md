# Design

## Context

RRG-62 gave the tool contract (`Tool`, `ToolResult`, `ToolRegistry` with JSON Schema validation and 30,000-character output truncation). RRG-64 added `src/nar/tools/files.py` (`resolve_path`, `is_binary`, `MAX_FILE_BYTES`, `os_error_message`) and `list_dir` with its `IGNORED_NAMES` set. RRG-63's agent exposes every registered tool to the model. Motivation: see `proposal.md`. Behavior: see `specs/glob-tool` and `specs/grep-tool`.

Constraints:
- Python 3.11+: `PurePath.full_match` (3.13) is not available, so glob patterns need their own matcher.
- No `ripgrep` on team machines and no new dependency wanted; `git` is installed wherever the team works on `nar`.
- Windows and Linux: paths shown to the model use `/`.

## Goals / Non-Goals

**Goals:**
- One file walker shared by both tools, so they always agree on what is ignored.
- Exactly git's ignore rules (nested `.gitignore`, `.git/info/exclude`, the global excludes file) without reimplementing them.
- Output that fits the registry's 30,000-character budget in normal use.

**Non-Goals:**
- Multiline regex, context lines around matches (`-A/-B`), counting-only modes. They can be added as optional parameters later.
- Sorting by modification time (alphabetical order is deterministic and testable).
- Following symlinked folders while walking (avoids cycles).

## Decisions

### 1. Layout
```
src/nar/tools/
├── search.py     NEW  walk_files(), glob_matcher(), display_path()
├── glob.py       NEW  GlobTool
├── grep.py       NEW  GrepTool
├── builtin.py    MODIFIED  registers GlobTool and GrepTool after ListDirTool
└── list_dir.py   unchanged (IGNORED_NAMES is imported from it)
src/nar/agent/prompt.py   MODIFIED  one guideline line about glob/grep
```

### 2. Which files exist: `walk_files(root) -> list[Path]`
1. Ask git: `git -C <root> ls-files -z --cached --others --exclude-standard` (tracked plus untracked-but-not-ignored files, paths relative to `root`, NUL-separated so any file name is safe; decoded with `os.fsdecode`). Timeout 30 s. Tracked files deleted from disk are dropped (`is_file()`).
2. If git is missing, `root` is not inside a repository (exit code 128), or the call fails or times out → fall back to `os.walk(root, followlinks=False)`, pruning folders named in `IGNORED_NAMES`.
3. In both cases, drop any path with a folder component in `IGNORED_NAMES` (a repo that commits `node_modules` still gets it skipped, matching `list_dir`), then sort by the relative POSIX path.
- **Why git over parsing `.gitignore`:** git's rules include negations, nested files, `info/exclude` and the user's global excludes; `git ls-files` applies all of them and is fast. **Alternative rejected:** the `pathspec` library — a new dependency and still our own walk-and-combine logic for nested files. **Alternative rejected:** requiring `ripgrep`.

### 3. Pattern matching: `glob_matcher(pattern) -> Callable[[str], bool]`
The pattern (leading `./` or `/` removed) is translated to one regular expression applied with `fullmatch` to a path relative to the searched folder:
- `**` as a whole segment: `(?:[^/]*/)*` in the middle or start, `.*` at the end;
- `*` → `[^/]*`, `?` → `[^/]`, `[abc]`/`[!abc]`/`[^abc]` → a character class (an unclosed `[` is literal), everything else escaped.
A pattern without `/` is matched against the file name only (any depth), so `*.py` and `test_*.py` do what the model expects. The same matcher serves `glob`'s `pattern` and `grep`'s `glob` option. Matching is case-sensitive on every OS.

### 4. `glob`
Schema: `pattern` (string, required, `minLength: 1`), `path` (string, optional); `additionalProperties: false`; `requires_approval = False`. Missing folder → `folder not found: <path>`; a file → `path is not a directory: <path>` (same wording as `list_dir`). Output: matching paths via `display_path` (relative to `context.cwd` when inside it, POSIX separators; otherwise absolute), one per line; over 200 → first 200 plus `[showing 200 of N files; use a narrower pattern or path]`; none → `(no files match <pattern>)`. `summary = "found N files"`.

### 5. `grep`
Schema: `pattern` (string, required, `minLength: 1`), `path`, `glob` (strings, optional), `ignore_case` (boolean, optional); `additionalProperties: false`; `requires_approval = False`.
- Compile first: `re.error` → `invalid regular expression <pattern>: <reason>`; nothing is read.
- `path` missing → `path not found: <path>`. A file → only that file (the `glob` option and ignore rules do not apply to an explicit file; binary or too large → error result, since the model asked for that file). A folder → `walk_files`, filtered by `glob` when given.
- Per file: skip if `stat().st_size > MAX_FILE_BYTES` or `is_binary(first 8 KiB)`; decode as UTF-8 with `errors="replace"`; split on `\n` (a trailing empty piece after a final newline is dropped; a `\r` at the end of a line is removed) so line numbers match editors (`splitlines()` would also split on form feeds); `regex.search(line)`. Unreadable files (`OSError`) are skipped.
- Output: `display_path:lineno:text`, text cut to 300 characters plus ` [...]`. Stop reading at the 101st match: first 100 plus `[stopped after 100 matches; narrow the pattern, path or glob]`. None → `(no lines match <pattern>)`. `summary = "N matches in M files"` (or `no matches`).
- 100 lines × (path + 300 characters) can exceed 30,000 characters only with very long lines and paths; then the registry's own truncation marker applies.
- **Why Python `re` and not a subprocess `grep`:** same syntax on Windows and Linux and no dependency on GNU grep.

### 6. System prompt
Add to `GUIDELINES` in `agent/prompt.py`: `- To find files or code, search with glob (file names) and grep (file contents) before reading files.` No spec change: it is wording that guides tool choice.

### 7. Tests
`tests/test_tools_search.py` (matcher table, `walk_files` with and without git, ignored folders, deleted tracked file), `tests/test_tools_glob.py`, `tests/test_tools_grep.py` with `tmp_path` projects; git-dependent tests create a repo with `git init` and are skipped when `git` is not on `PATH`. `tests/test_tools_registry.py`'s list of built-in names gains `glob` and `grep`.

## Risks / Trade-offs

- [`git` missing on a machine] → Fallback walk still skips noise folders; only `.gitignore` rules are lost. Stated in the README.
- [Huge repositories make `grep` slow (pure Python)] → Binary and >10 MB files are skipped and the match cap stops early; `path`/`glob` narrow the search. Acceptable for course-sized projects.
- [Pattern-without-slash rule differs from shell globbing] → It matches `.gitignore` and ripgrep behavior and is stated in the tool description.
- [Case-sensitive matching on Windows/macOS file names] → Deterministic across OSes; the model can use `[Rr]eadme*` if needed.

## Migration Plan

No new dependency, configuration or data. `pip install -e ".[dev]"` is unchanged. Rollback: revert the PR; no other tool depends on these.
