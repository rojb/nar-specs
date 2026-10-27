# Design

## Context

After RRG-62 the `nar` package has the tool framework: `src/nar/tools/` (`Tool`, `ToolContext`, `ToolResult` in `base.py`; `ToolRegistry` in `registry.py`; `validation.py`; `limits.py`; `ReadFileTool` in `read_file.py`; `register_builtin_tools()` / `create_default_registry()` in `builtin.py`), `src/nar/ui/tool_view.py` (`ToolRenderer`, `summarize_arguments`) and the runner `scripts/run_tool.py`. Motivation and scope: see `proposal.md`. Required behavior: see `specs/write-file-tool`, `specs/edit-file-tool`, `specs/list-dir-tool` and the `tool-framework` delta (tool activity display).

Constraints:
- Python 3.11+, Windows (PowerShell, cp1252 consoles) and Linux/macOS. No new dependency: `difflib`, `os`, `re` are standard library; `rich` is already a dependency.
- `ToolRegistry.execute` is the single choke point: it validates arguments, catches exceptions and truncates `output` to `MAX_OUTPUT_CHARS`. Its `_finish` currently rebuilds the `ToolResult` copying three fields by hand, so any new result field must be preserved there.
- RRG-61 adapters (and RRG-63 later) read only `ToolResult.output` and `is_error`; nothing provider-side may depend on a new field.
- `read_file` shows every line break as LF (it strips `\r\n`, and also splits on a lone `\r`), so the model builds `old_string` with LF. This is the reason for the line-break rule of `edit_file`.
- RRG-69 (approval) reads `requires_approval` and wants a diff to show; RRG-72 (hooks) reacts to `edit_file`. Neither is implemented here.

## Goals / Non-Goals

**Goals:**
- Three tools that follow the existing contract unchanged (`Tool` subclass, error results instead of exceptions, bare error messages that include the resolved path).
- `edit_file` never corrupts a file: every detectable failure happens before the first byte is written, and untouched bytes stay identical, including mixed line endings.
- The diff reaches both the model (inside `output`) and the terminal (as structured data), without changing how existing tools or provider adapters behave.

**Non-Goals:**
- Atomic write via temp file + rename, backups or undo (checkpoints belong to the session change).
- Confining paths to the project folder, approval prompts (RRG-69).
- Detecting file encodings other than UTF-8, handling lone-CR (classic Mac) line endings, or editing several occurrences at once (no `replace_all`).
- Recursive listing, glob filters, `.gitignore` parsing for `list_dir`, or configurable ignore lists.
- A diff for `write_file` (the specs do not ask for one; see Decision 7).

## Decisions

### 1. Module layout
```
src/nar/
├── tools/
│   ├── base.py          MODIFIED  ToolResult gains `diff: str | None = None`; ok() takes diff
│   ├── files.py         NEW       shared helpers: resolve_path, is_binary, os_error_message,
│   │                              MAX_FILE_BYTES, BINARY_SNIFF_BYTES (moved from read_file.py)
│   ├── read_file.py     MODIFIED  uses files.resolve_path and the moved constants; behavior identical
│   ├── write_file.py    NEW       WriteFileTool
│   ├── edit_file.py     NEW       EditFileTool (+ private line-break helpers and diff builder)
│   ├── list_dir.py      NEW       ListDirTool, IGNORED_NAMES
│   ├── registry.py      MODIFIED  ToolRegistry._finish keeps `diff` (dataclasses.replace)
│   ├── builtin.py       MODIFIED  register_builtin_tools(): + WriteFileTool, EditFileTool, ListDirTool
│   └── __init__.py      unchanged (built-in tools are not re-exported, as with ReadFileTool)
└── ui/
    └── tool_view.py     MODIFIED  ToolRenderer.show_result renders result.diff
scripts/run_tool.py      unchanged (see Decision 3)
pyproject.toml           unchanged (no new dependency)
tests/test_tools_base.py            MODIFIED  diff defaults to None; ok(diff=...)
tests/test_tools_registry.py        MODIFIED  default registry now lists four tools; diff survives truncation
tests/test_tool_view.py             MODIFIED  diff rendering cases
tests/test_tools_files.py           NEW
tests/test_tools_write_file.py, test_tools_edit_file.py, test_tools_list_dir.py   NEW
README.md                MODIFIED  short section on the three tools
```
`nar/errors.py`, `nar/cli/*`, `nar/config/*`, `nar/providers/*` are **not** touched. **New config keys: none** (neither `~/.nar/config.toml` nor `.nar/config.toml`).

### 2. How the diff travels: an optional `diff` field AND the text inside `output`
```python
@dataclass(frozen=True)
class ToolResult:
    output: str
    is_error: bool = False
    summary: str | None = None
    diff: str | None = None          # unified diff for the terminal; None for most tools
    # ok(output, summary=None, diff=None); error(message) unchanged
```
`edit_file` returns `output = "<summary line>\n\n<diff>"` and `diff = <diff>`. The model therefore sees the diff (spec: "so the model and the user can see exactly what changed") through `output`, which is all RRG-61 reads; the terminal gets the same text as structured data it can colorize without parsing `output`.
- **Backward compatibility:** the new field is last and defaults to `None`, so every positional/keyword construction in code and tests keeps working, and dataclass equality (`ToolResult(output="hi", summary="echoed")`) is unchanged for results without a diff. RRG-61 adapters ignore it.
- **Truncation:** `truncate_output` applies to `output` only, so the model-facing copy stays capped at 30,000 characters; the terminal `diff` is not capped (it is bounded by the strings the model sent plus 3 context lines per hunk). `ToolRegistry._finish` must carry `diff` over when it rebuilds the result; it switches to `dataclasses.replace(result, output=output)` so future fields cannot be dropped again.
- **Alternatives rejected:** (a) diff only in `output`, with the renderer detecting `--- `/`+++ ` markers: fragile (a `read_file` of a `.diff` file would be colorized) and breaks the spec scenario "No diff for results without one". (b) A separate `metadata: dict` on the result: speculative; a typed `diff` is all RRG-69 and the specs need. (c) Return the diff only in the field: the model would not see it.

### 3. Rendering the diff (`ui/tool_view.py`)
`show_result` keeps its current output (indicator + summary / error message) and, for a **successful** result with `diff`, prints the diff below it through a new private `_print_diff(diff)`. Errors never print a diff.
- Each diff line becomes a `rich.text.Text` with a style chosen by its first characters: `+++`/`---` file headers bold, `@@` hunk headers cyan, `+` green, `-` red, anything else (context) unstyled. Header lines are tested before `+`/`-` so `--- a/config.py` is not painted as a removal.
- `Text` is used instead of markup strings: it is never parsed, so `[red]` inside a diff is printed literally (spec scenario "Diff text is shown literally") without relying on `escape`. Existing lines still use `escape`.
- **Non-UTF consoles:** when `console.encoding` is not UTF-based (same test as `_indicator`), each line is passed through `line.encode(encoding, "replace").decode(encoding)`, so a diff containing characters outside cp1252 prints `?` instead of raising `UnicodeEncodeError`. The `+`/`-` prefixes and colors already make added/removed distinct without any glyph.
- Printed with `soft_wrap=True, highlight=False`, like the existing lines, so long lines are not re-wrapped and Rich's highlighter does not recolor numbers.
- `scripts/run_tool.py` is unchanged: it renders through `ToolRenderer` and then echoes `result.output`, so the developer sees the colored diff and, below it, the exact text the model would receive (the diff appears twice by design; the second copy is the model view).
- **Alternative rejected:** `rich.syntax.Syntax(diff, "diff")`: pulls Pygments theme colors that differ per terminal, colors the `---`/`+++` header lines like removals, and makes assertions on styles harder.

### 4. Shared file helpers (`tools/files.py`) and the `read_file` refactor
New module with only what at least two tools need:
```python
MAX_FILE_BYTES = 10 * 1024 * 1024        # moved from read_file.py
BINARY_SNIFF_BYTES = 8192                # moved from read_file.py
def resolve_path(raw: str, cwd: Path) -> Path      # Path(raw); relative -> cwd / raw; .resolve(); no ~ expansion
def is_binary(head: bytes) -> bool                 # b"\x00" in head[:BINARY_SNIFF_BYTES]
def os_error_message(action: str, path: Path, exc: OSError) -> str   # f"cannot {action} {path}: {exc.strerror or exc}"
```
- `read_file.py` imports `resolve_path`, `is_binary`, the constants and `os_error_message`, and still re-exports `MAX_FILE_BYTES` / `DEFAULT_LIMIT` (the existing test imports `MAX_FILE_BYTES` from `nar.tools.read_file`). Its messages (`file not found: <path>`, `path is a directory: <path>`, too large, binary, `cannot read <path>: ...`) and the two-step check order are not changed; the full existing `test_tools_read_file.py` must pass untouched, which is the acceptance check for the refactor. If any message would change, the duplicate is kept instead.
- `write_file`, `edit_file` and `list_dir` use `resolve_path` and `os_error_message` (`cannot write`, `cannot read`, `cannot list`). The exists/is-directory checks are not shared: their wording differs by tool (`file not found` vs `folder not found`, `path is a directory` vs `path is not a directory`) and each is two lines.
- **Alternative rejected:** a `_check_readable` shared by read and edit: read's too-large message tells the model to "use a narrower tool", wrong advice for an edit, so the shared piece stays at the primitive level.

### 5. `edit_file` (`tools/edit_file.py`, `EditFileTool`)
Schema: `path`, `old_string`, `new_string` (all `string`, all required), `additionalProperties: false`. `requires_approval = True`. Description tells the model that `old_string` must match exactly once, so it should include surrounding lines, and that line breaks can be written as `\n`.

Flow (every failure is `ToolResult.error`; nothing is written until step 7):
1. **Request checks, no I/O:** normalize `old_string` and `new_string` with `replace("\r\n", "\n")`. Empty `old_string` → `old_string must not be empty; use write_file to create or replace whole files`. Normalized `old_string == new_string` → `old_string and new_string are identical; nothing to change`. (Comparing normalized text means a request that differs only in line-break style is also "no change".)
2. **Path:** `resolve_path`; not found → `file not found: <path>` (never created); directory → `path is a directory: <path>`; size over `MAX_FILE_BYTES` → `file is too large (<n> MB, limit 10 MB) to edit` (guard so an edit cannot load an unbounded file; not in the specs).
3. **Read bytes** once (`path.read_bytes()`), so no newline translation happens on Windows. `is_binary(data)` → `file appears to be binary: <path>`. `data.decode("utf-8")` strict; `UnicodeDecodeError` → `file is not valid UTF-8: <path>`. A UTF-8 BOM decodes to `﻿`, stays in `text` and is written back untouched.
4. **Match on the raw text with a line-break-tolerant pattern**, not on a normalized copy: `pattern = r"\r?\n".join(re.escape(part) for part in old.split("\n"))`, `matches = list(re.finditer(pattern, text))`. A line break in `old_string` matches `\n` or `\r\n` in the file; everything else is literal and case/whitespace-exact. Matching on the raw text means each match has real start/end offsets, so no mapping from a normalized copy back to the original is needed, and mixed-ending files work without special cases.
5. **Unique-match rule:** 0 matches → `old_string was not found in <path>`; more than 1 → `old_string was found <n> times in <path>; include more surrounding context to make it unique`.
6. **Replacement text and style:** `CRLF = text.count("\r\n")`, `LF = text.count("\n") - CRLF`. The file style is CRLF only when `CRLF > LF`; a file without line breaks, or a tie, uses LF. `replacement = normalized new_string`, with `"\n"` replaced by `"\r\n"` when the style is CRLF. Only the `[start:end]` of the single match is replaced: `new_text = text[:m.start()] + replacement + text[m.end():]`, concatenation rather than `re.sub` so backslashes in `new_string` are never interpreted.
7. **Write:** `path.write_bytes(new_text.encode("utf-8"))`. `OSError` (read-only file, permissions) at read or write → `cannot read|write <path>: <os reason>`.

**Mixed line endings (explicit rule):** bytes outside the matched span are never rewritten, so a file with both CRLF and LF keeps each existing line break as is. A multi-line `old_string` matches across any mixture of `\n`/`\r\n` between its lines (the whole matched span is replaced). The only place a style is chosen is the new text, which follows the file's majority style, tie → LF. A lone `\r` is not a line break for `edit_file`: it is matched literally (documented limitation; `read_file` does show it as a line break, but the spec's contract is LF/CRLF).

**Diff and summary:** `before = text.replace("\r\n", "\n")` and `after = new_text.replace("\r\n", "\n")` are split on `"\n"` (dropping the single empty last element produced by a trailing newline), so CRLF noise never appears in the diff and a CRLF file produces the same diff as its LF twin. `difflib.unified_diff(before, after, fromfile=f"a/{shown}", tofile=f"b/{shown}", n=3, lineterm="")`, where `shown` is the path relative to `context.cwd` when inside it, otherwise the resolved path, in forward slashes. This gives the spec's header with the path, `-`/`+` lines and 3 unchanged context lines per hunk. Added/removed counts come from the diff body (lines starting with `+`/`-` after the two header lines), summary `edited <path.name> (+A -R)`. `output = f"edited {path} (+A -R)\n\n{diff}"`, `diff = diff`.

- **Why regex over `text.replace` on a normalized copy:** the normalize-then-map-back approach needs an offset table for every `\r\n` and is the usual source of off-by-one corruption; the regex gives exact offsets, and `re.escape` on each LF-separated part keeps the pattern fully literal. Cost is negligible under the 10 MB cap.
- **Why not `str.count` on raw text:** a CRLF file never contains the LF-only `old_string`, so the model's edits would fail with "not found" on Windows checkouts.
- **Non-overlapping counting:** `finditer` counts non-overlapping matches like `str.count`, which is what "appears N times" means for the model.
- **Atomic write (temp file + `os.replace`) rejected:** on Windows `os.replace` fails when the target is read-only or open, it replaces symlinks and hard links with new files, drops permissions/ACLs unless copied, and can leave stray temp files, in exchange for protection against a crash during a write of a file that is at most 10 MB and was just fully read and validated. All detectable errors already occur before the write, so a partial file requires a crash or full disk in that instant. Recorded as a risk; a checkpoint system (sessions change) is the better answer for undo.

### 6. `write_file` (`tools/write_file.py`, `WriteFileTool`)
Schema: `path` and `content` (strings, both required), `additionalProperties: false`. `requires_approval = True`.
- **Content is written exactly as given:** `data = content.encode("utf-8")` then `path.write_bytes(data)`. `write_bytes` performs no newline translation (text mode would turn `\n` into `\r\n` on Windows, contradicting the spec's "no newline conversion"). A `content` that cannot be encoded (lone surrogates from a bad JSON escape) raises `UnicodeEncodeError`; it is caught before touching the filesystem and returned as `content cannot be encoded as UTF-8`.
- **Checks before any filesystem change:** resolve path; `is_dir()` → `path is a directory: <path>`. Then walk from `path.parent` upward to the first ancestor that exists; if it is not a directory → `a parent of the path is not a folder: <ancestor>` (this is the `app.py/x.txt` scenario: `Path.exists()` is false for it, so only the ancestor walk detects it, and no folder is created).
- **Write:** `existed = path.exists()` (before), `path.parent.mkdir(parents=True, exist_ok=True)`, `path.write_bytes(data)`. `OSError` → `cannot write <path>: <os reason>`.
- **Report:** lines = number of `"\n"` plus 1 when non-empty content does not end with `"\n"` (`"hello\n"` is 1 line, `""` is 0, `"a\nb"` is 2); bytes = `len(data)`. `output = "created <path> (<n> lines, <b> bytes)"` or `"overwritten ..."`; `summary = "created notes.txt (3 lines)"` / `"overwrote notes.txt (3 lines)"` using `path.name`.
- No `diff` on overwrite: the specs and the proposal's "Done when" only require a diff for `edit_file`, and a whole-file diff would double the tokens of every overwrite sent to the model. The `diff` field exists, so RRG-69 can ask for it later without changing the contract.
- **Alternative rejected:** `open(path, "w", encoding="utf-8")`, which translates `\n` on Windows; and `newline=""` text mode, which works but needs the encode step anyway for the error case.

### 7. `list_dir` (`tools/list_dir.py`, `ListDirTool`)
Schema: optional `path` (string), no `required`, `additionalProperties: false`. `requires_approval = False`. `IGNORED_NAMES = frozenset({".git", "node_modules", ".venv", "__pycache__", ".pytest_cache", ".ruff_cache", ".mypy_cache"})`.
- **Path:** `resolve_path(arguments.get("path", "."), context.cwd)`. Not exists → `folder not found: <path>`; not a directory → `path is not a directory: <path>`.
- **Listing:** `with os.scandir(path) as entries:` collect `(name, is_dir)` for entries whose `name` is not in `IGNORED_NAMES`. The ignore rule is on entry names only, so `list_dir(path="node_modules")` lists its contents. `scandir` (not `iterdir`) gives `is_dir()` from the directory read without an extra `stat` per entry. `is_dir()` follows symlinks, so a symlink to a folder shows as `name/`; a broken symlink or an entry whose `is_dir()` raises `OSError` is listed as a file. Hidden entries other than the ignored ones are kept.
- **Order and format:** sorted by `(name.casefold(), name)` (case-insensitive per spec, case-sensitive tie-break so the output is deterministic), folders and files interleaved, one entry per line, folders suffixed `/`. `pyproject.toml`, `README.md`, `src/` follows from `casefold`.
- **Empty:** no entries after filtering → success with output `(folder is empty)`, summary `listed 0 entries` (same style as `read_file`'s `(file is empty)`).
- **Summary:** `listed N entries`. Large folders are bounded by the registry's 30,000-character cap with its truncation marker; no tool-specific limit.
- `OSError` from `scandir` (permission) → `cannot list <path>: <os reason>`.
- **Alternatives rejected:** `Path.iterdir()` plus `is_dir()` (one `stat` per entry); `os.walk`/recursion and `.gitignore` support (out of spec); an `ignore` argument for the model (speculative).

### 8. Registration, export and approval flag
`register_builtin_tools` registers, in this order: `ReadFileTool`, `WriteFileTool`, `EditFileTool`, `ListDirTool` (reading first, then the video's order). The order is the export order, so tests assert it exactly. `requires_approval` is `True` on write/edit and `False` on list; the flag stays out of the exported definition, as designed in RRG-62, so the "Listed in the export" scenarios check `name`, `description` and a `parameters` schema whose `required` is `["path", "content"]`, `["path", "old_string", "new_string"]` and absent/empty. `tools/__init__.py` is unchanged.

### 9. Tests
Offline `pytest` with `tmp_path` and `ToolContext(cwd=tmp_path)`, same style as `test_tools_read_file.py` (a `run(context, **arguments)` helper calling `Tool().execute` directly). File fixtures are written and asserted as **bytes** (`write_bytes` / `read_bytes`) so CRLF behavior is real on Windows and Linux alike.
- `test_tools_write_file.py`: every spec scenario (create, overwrite, relative path, empty content, parents created, directory, parent-is-file with nothing created, read-only file via `chmod`/skipped where the OS ignores it, `"a\n"` stays `b"a\n"` and `"a\r\nb"` stays CRLF byte-for-byte), report texts, `requires_approval`.
- `test_tools_edit_file.py`: unique replace, relative path, multi-line, case-exact miss, deleting text, empty/identical `old_string`, not found, three matches (count in the message, file bytes unchanged), directory, missing file (and not created), PNG header as binary, `b"\xff\xfe..."` as non-UTF-8, CRLF file with a one-line edit (all CRLF kept), CRLF file with an LF multi-line `old_string` (replacement written as CRLF), LF file with CRLF in `old_string`, mixed-ending file (untouched lines byte-identical, tie → LF), BOM preserved, backslashes in `new_string` literal, diff header/`-`/`+`/context lines and `(+1 -1)` summary, `output` contains the diff, `diff` field set.
- `test_tools_list_dir.py`: default path via context cwd, sorting with mixed case, `/` suffix, non-recursive, each ignored name skipped, explicit path inside an ignored folder, empty folder, only-ignored folder, not found, file path, a symlink case skipped when the OS cannot create symlinks.
- `test_tools_files.py`: `resolve_path` relative/absolute, `is_binary`, `os_error_message`.
- `test_tool_view.py` (extended): green/red/uncolored diff lines using a `Console(file=StringIO(), force_terminal=True, color_system="standard")` and asserting ANSI codes around the `+`/`-` lines and none around context; `[red]` inside a diff printed literally; no diff section for `read_file`-style results; cp1252 console with a non-cp1252 character in the diff does not raise.
- `test_tools_registry.py` / `test_tools_base.py` (updated): default registry names are `["read_file", "write_file", "edit_file", "list_dir"]` and the export has exactly three fields per tool; `diff` defaults to `None`; a long result keeps `diff` after truncation.
- Manual checklist in `tasks.md`: `run_tool.py write_file` then `edit_file` then `list_dir` in a real PowerShell terminal, plus a CRLF file created on Windows.

### 10. README
`README.md` gets a short "Built-in tools" addition next to the existing `scripts/run_tool.py` paragraph: the four tools, which two require approval (enforced by RRG-69, not yet asked), the line-break rule of `edit_file`, and an example `write_file` then `edit_file` run.

## Risks / Trade-offs

- [Non-atomic write: a crash or full disk during `write_bytes` could leave `edit_file`/`write_file` targets truncated] → All validation and matching happen before the write; the window is one `write_bytes` call on a file of at most 10 MB. Checkpoints/undo are a session-change concern; revisit atomic writes there if it proves a real problem.
- [Both tools accept any path, including outside the project, and `write_file` overwrites without a backup] → Out of scope per the proposal; `requires_approval` is set so RRG-69 can show the path and diff before running. Recorded for that change.
- [A file with mixed line endings gets new text in the majority style, so the style inside it can drift] → Existing bytes are never rewritten; the tie rule is deterministic and documented; mixed files are already inconsistent.
- [A lone `\r` (classic Mac) file: `read_file` shows lines, but `edit_file` multi-line `old_string` will not match across them] → Single-line edits still work; unsupported by the spec's LF/CRLF contract, so reported as "not found" and the model can fall back to `write_file`.
- [Matching 0 or >1 times is an error even when the model's intent is obvious] → This is the specified behavior; the messages state the count and ask for more context.
- [A very large `old_string` builds a long regex] → `re.escape`d literal parts joined by `\r?\n` have no backtracking hazards; bounded by the 10 MB file cap and the model's output size.
- [Diff in `output` consumes model tokens, and `_finish` truncates it at 30,000 characters] → A diff is bounded by the strings the model sent; the terminal copy in `diff` is complete. The truncation marker tells the model when it was cut.
- [Registry `_finish` silently dropping new `ToolResult` fields] → Switched to `dataclasses.replace`, with a regression test.
- [Symlinked folders appear as folders, and a symlink loop is not followed because listing is not recursive] → No special handling required.
- [`list_dir` hides `.git`, `.venv`, ... even when the user wants to see them] → Pass the folder as `path` (spec); no config key in this change.

## Migration Plan

No data, config or CLI migration. `pip install -e ".[dev]"` is unchanged (no new dependency). Behavior of `read_file`, the registry export of `read_file` and every existing `ToolResult` construction is unchanged; the new tools and the `diff` field are additive. Rollback: revert the PR; nothing else depends on the new modules until RRG-63 exposes the registry to the model and RRG-69 reads the approval flag.

## Open Questions

- Whether RRG-69 wants `write_file` to attach a diff when overwriting (it would show the user what is lost). It can be added then by filling the existing `diff` field and does not affect any spec or task here.
