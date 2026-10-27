# Design

## Context

After RRG-60 the `nar` package has `cli/`, `config/` (`NarConfig`, loader) and `context/`, a `NarError` hierarchy in `nar/errors.py` (`NarError`, `ConfigError`) and an error boundary in `nar/cli/main.py` that prints expected errors as one line. There is no `tools/` or `ui/` package yet. Motivation and scope: see `proposal.md`. Required behavior: see `specs/tool-framework` and `specs/read-file-tool`.

Constraints:
- Python 3.11+, Windows (PowerShell) and Linux/macOS, plain `pip` (same as RRG-60). `pydantic>=2` and `click` are already dependencies.
- RRG-61 (`add-llm-providers`, nathaliascode) runs in parallel. Agreed 2026-10-18: the registry exports provider-neutral definitions `{name, description, parameters}`; RRG-61 converts them to Anthropic/OpenAI formats. RRG-61's design takes tool definitions as `Sequence[Mapping[str, Any]]`, receives calls as `ToolCall(id, name, arguments: dict)`, and builds the reply with `Message.tool_result(call_id, text, is_error=False)`. RRG-62 depends only on RRG-60 (proposal "Depends on"), so **no RRG-62 module imports `nar.providers`**; the two meet in RRG-63 through plain dicts and strings.
- Later consumers fix what must stay extensible: RRG-63 (agent loop calls the registry), RRG-64/65/66/67 (more built-in tools), RRG-69 (reads `requires_approval`), RRG-72 (hooks around tool calls), RRG-74 (MCP tools with schemas that arrive as raw JSON Schema), RRG-75 (user-defined `Tool` subclasses loaded from disk).

## Goals / Non-Goals

**Goals:**
- One small contract (`Tool`, `ToolResult`, `ToolRegistry`) that the loop and every later tool use without knowing each other.
- A tool call never raises into the caller for a reason the model can fix (unknown tool, bad arguments, tool failure); it returns an error result.
- Tools testable offline with `pytest` and runnable without an LLM (`scripts/run_tool.py`).

**Non-Goals:**
- Async tools, parallel tool execution, per-tool timeouts. The CLI and RRG-61 are synchronous; tools that can hang (shell, RRG-65) own their timeout.
- Approval prompts (RRG-69), hooks (RRG-72), MCP/custom tool loading (RRG-74/75). Only the extension points listed under Decision 9 exist here.
- Confining `read_file` to the working directory. The spec allows any absolute path; see Risks.
- Configuration for limits: output cap, line default and size cap are fixed by the specs and live as constants.
- Calling the renderer from the registry; the caller does that (Decision 8).

## Decisions

### 1. Package layout
```
src/nar/
├── errors.py                 MODIFIED  + ToolError(NarError)
├── tools/                    NEW
│   ├── __init__.py           NEW  re-exports Tool, ToolContext, ToolResult, ToolDefinition,
│   │                              ToolRegistry, create_default_registry
│   ├── base.py               NEW  Tool (ABC), ToolContext, ToolResult, ToolDefinition, NAME_PATTERN
│   ├── validation.py         NEW  validate_arguments(schema, arguments) -> list[str]
│   ├── limits.py             NEW  MAX_OUTPUT_CHARS, truncate_output()
│   ├── registry.py           NEW  ToolRegistry
│   ├── read_file.py          NEW  ReadFileTool
│   └── builtin.py            NEW  register_builtin_tools(), create_default_registry()
└── ui/                       NEW
    ├── __init__.py           NEW  re-exports ToolRenderer
    └── tool_view.py          NEW  ToolRenderer, summarize_arguments()
scripts/run_tool.py           NEW  developer runner
pyproject.toml                MODIFIED  + rich, jsonschema in dependencies
tests/test_tools_base.py, test_tools_registry.py, test_tools_read_file.py,
      test_tool_view.py, test_run_tool_script.py     NEW  offline pytest
README.md                     MODIFIED  one paragraph on scripts/run_tool.py
```
`nar/cli/main.py`, `nar/config/*` and `nar/context/*` are **not** touched: tool activity rendering and registry wiring become visible only when RRG-63 connects the loop. **New config keys: none** (neither `~/.nar/config.toml` nor `.nar/config.toml`).

### 2. `Tool` is an ABC with declarative attributes; the schema is a plain JSON Schema dict
```python
class Tool(ABC):
    name: str
    description: str
    parameters: dict[str, Any]          # JSON Schema, type "object"
    requires_approval: bool = False

    @abstractmethod
    def execute(self, arguments: dict[str, Any], context: ToolContext) -> ToolResult: ...
    def definition(self) -> ToolDefinition  # {"name", "description", "parameters"} deep-copied
```
`ToolContext` is a frozen dataclass (`cwd: Path`); `ToolDefinition` is a `TypedDict` with exactly `name`, `description`, `parameters`. Because it is a plain `dict`, it already satisfies RRG-61's `Mapping[str, Any]` input.

- **Why an ABC:** the roadmap and RRG-75 load *subclasses of `Tool`* from `.nar/tools/`; an ABC gives a clear error for a missing `execute` and one obvious thing to subclass. Declaring `name`/`description`/`parameters` as attributes also lets RRG-74 set them per instance for MCP tools whose schema arrives at runtime. **Alternative rejected:** `typing.Protocol`. Structural typing gives no instantiation-time check and nothing for RRG-75 to inherit from.
- **Why a raw dict schema, not a pydantic model:** MCP (RRG-74) and custom tools already hold raw JSON Schema, so validation must work on a dict anyway. A pydantic model would add a second path (and its `title`/`$defs` noise would leak into the schema sent to the model) for one built-in tool. **Alternative rejected:** `model_json_schema()` for built-ins only; two sources of truth for the same contract.
- **Why `context` is a per-call argument and not a constructor argument:** tools stay zero-argument constructible (needed by RRG-75's loader) and tests can pass a `tmp_path` context. It is the single place later changes can add per-call data without changing every tool's constructor.
- `requires_approval` is a class attribute defaulting to `False`. It is deliberately **not** part of the exported definition: providers do not need it and the spec fixes the export to three fields. The "Tool exposes all fields" scenario is verified on the `ReadFileTool` instance (`name`, `description`, `parameters`, `requires_approval is False`).

### 3. Results: `ToolResult` returned, `ToolError` raised only for programmer mistakes
```python
@dataclass(frozen=True)
class ToolResult:
    output: str                  # what the model sees (success text or bare error message)
    is_error: bool = False
    summary: str | None = None   # optional one-liner for the terminal, e.g. "read 5 lines"
    # classmethods: ToolResult.ok(output, summary=None), ToolResult.error(message)
```
- The model-facing mapping for RRG-63 is one line: `Message.tool_result(call.id, result.output, is_error=result.is_error)` (RRG-61 Decision 3). RRG-62 never imports that type.
- Error `output` is the **bare message** without an `Error:` prefix, because RRG-61's OpenAI-compatible adapter already adds `Error: ` when `is_error` is true; a prefix here would double it.
- `ToolError(NarError)` (in `nar/errors.py`, **modified**) is raised for *registration/definition* mistakes (invalid name, duplicate name, schema that is not a valid object schema, lookup of an unknown tool through `get()`). These are bugs in `nar` or a custom tool, so the CLI error boundary shows them as `error: ...` (NarError path). A tool's `execute()` may also `raise ToolError("...")` for an expected failure; the registry turns it into `ToolResult.error(message)`.
- `execute()` calls made through the registry never raise for model- or tool-caused failures (Decision 5). `KeyboardInterrupt` is not caught.
- **Alternative rejected:** tools raise for every failure and the loop converts. That spreads try/except over each caller (loop, script, subagents, hooks) and makes "nar keeps running" depend on every caller getting it right.

### 4. Argument validation with `jsonschema` (`tools/validation.py`)
`validate_arguments(schema, arguments) -> list[str]` runs `jsonschema.Draft202012Validator(schema).iter_errors(arguments)` and rewrites each error into a model-readable line naming the argument:

| jsonschema validator | Message |
|---|---|
| `required` (one per missing name) | `missing required argument 'path'` |
| `type` | `argument 'offset' must be of type integer, got string` |
| `additionalProperties` | `unknown argument 'foo'` |
| anything else (`minimum`, `enum`, ...) | `argument 'offset': 0 is less than the minimum of 1` |

An empty list means valid. No type coercion (`"10"` is not accepted for an integer), so the model learns the real contract. Registration calls `Draft202012Validator.check_schema(parameters)` and requires `parameters["type"] == "object"`, so a broken schema fails at startup instead of on the first call.
- **Why `jsonschema` (confirms the proposal's open choice):** it validates the same raw schemas that built-in, MCP and custom tools carry and it is the reference implementation. **Alternatives rejected:** a hand-written type checker (re-implements `required`/`type`/`enum`/`minimum` badly and diverges from what providers enforce); pydantic `TypeAdapter` (cannot validate against a raw JSON Schema).

### 5. `ToolRegistry` (`tools/registry.py`)
```python
class ToolRegistry:
    def __init__(self, context: ToolContext | None = None)   # default: ToolContext(cwd=Path.cwd())
    def register(self, tool: Tool) -> None        # ToolError: invalid name / duplicate / bad schema
    def get(self, name: str) -> Tool              # ToolError: "tool 'x' does not exist; available: a, b"
    def names(self) -> list[str]                  # registration order
    def definitions(self) -> list[ToolDefinition] # registration order, exactly 3 fields each
    def execute(self, name: str, arguments: Mapping[str, Any]) -> ToolResult   # never raises
```
- Backed by a `dict[str, Tool]` (insertion order = export order). Names are validated against `NAME_PATTERN = ^[a-zA-Z][a-zA-Z0-9_]{0,63}$` from the spec; the error names the offending tool. A duplicate raises before touching the dict, so the original tool stays.
- `execute()` pipeline: lookup (unknown tool → `ToolResult.error` with the same message as `get()`, which lists the available names) → `validate_arguments` (non-empty → `ToolResult.error("Invalid arguments for <tool>:\n- ...")`, tool not run) → `tool.execute(dict(arguments), context)` inside `try/except` (`ToolError` → its message; any other `Exception` → `"<ExceptionType>: <message>"`) → `truncate_output` (Decision 6). `arguments` comes straight from RRG-61's `ToolCall.arguments`; RRG-61 maps unparseable JSON to `{}`, which this pipeline reports as missing required arguments.
- **Instance, not a global singleton:** tests build isolated registries and RRG-73 subagents can later get a registry with a subset of tools. `create_default_registry(cwd)` is the production entry point.
- **Alternative rejected:** a module-level registry filled by a `@register_tool` decorator at import time (implicit ordering, global state in tests, import side effects that RRG-75's dynamic loading would have to work around).

### 6. Output truncation (`tools/limits.py`)
`MAX_OUTPUT_CHARS = 30_000`. `truncate_output(text)` returns `text` unchanged when short; otherwise it keeps the head and appends `\n[output truncated: N characters omitted]`, sized so that **head plus marker never exceeds the limit** (a fixed reserve for the marker is subtracted from the head). That is why the spec says "20,000 **or more** characters omitted" for 50,000 characters of input. It is applied in `ToolRegistry.execute` to every result (success and error), so MCP and custom tools get the cap for free and individual tools never implement it.

### 7. `read_file` (`tools/read_file.py`, `ReadFileTool`)
Schema: `path` (string, required), `offset` (integer, `minimum: 1`), `limit` (integer, `minimum: 1`), `additionalProperties: false`. `requires_approval = False`. Constants: `DEFAULT_LIMIT = 2000`, `MAX_FILE_BYTES = 10 * 1024 * 1024`, `BINARY_SNIFF_BYTES = 8192`.

Flow (each failure is a `ToolResult.error`, never an exception):
1. **Path:** `Path(path)`; if relative, join with `context.cwd`; `.resolve()`. The resolved path appears in every error message (spec: "including the resolved path"). No `~` expansion (not in the spec).
2. **Checks in order:** does not exist → `file not found: <path>`; `is_dir()` → `path is a directory: <path>`; `stat().st_size > MAX_FILE_BYTES` → `file is too large (<n> MB, limit 10 MB); use a narrower tool such as a search instead of reading the whole file`; first 8 KiB contains a NUL byte → `file appears to be binary: <path>`; `OSError` (permissions, etc.) → its message.
3. **Read:** open as text, `encoding="utf-8", errors="replace"` (non-UTF-8 text is returned with replacement characters, per spec), iterate the file once: lines before `offset` are only counted, lines in range are kept, the rest are only counted. One pass gives the total line count that the spec requires in the partial-read message, and the 10 MB cap bounds the cost. Newlines are split by the file iterator (`\n`, `\r\n`, `\r`) and stripped.
4. **Range errors:** `offset` greater than the total (file not empty) → `offset <n> is past the end of the file; the file has only <total> lines`. An empty file is a success with output `(file is empty)`.
5. **Format:** each line as `{lineno:>W}\t{text}` where `W` is the width of the last shown line number. When the range does not cover the whole file, a first line states `[Showing lines 10-14 of 100]` (plus `use offset=15 to continue` when more remain). It is placed **first** so that registry truncation (Decision 6) can never cut it off. `summary` = `read N lines` (the terminal summary the spec asks for).
- **Why only a NUL-byte sniff for binary:** it is the same heuristic git and grep use, costs 8 KiB, and catches PNG/ZIP/executables; UTF-16 text is also (acceptably) reported as binary.
- **Why the one-pass reader over `read_text().splitlines()`:** avoids holding a 10 MB file as a list of strings for a 5-line request, and `splitlines()` also splits on form-feed and other separators, which would shift line numbers versus editors.

### 8. Terminal display (`ui/tool_view.py`, `ToolRenderer`) and Rich
`ToolRenderer(console: rich.console.Console | None = None)` with two calls: `show_call(name, arguments)` prints `-> read_file(path="pyproject.toml", limit=5)`; `show_result(name, result)` prints a green success indicator plus `result.summary` (fallback: `<n> lines, <m> chars` from the output) or a red error indicator plus `result.output`. `summarize_arguments(arguments, max_len=100)` renders `key="value"` pairs, collapsing newlines and shortening long values with `...`, so a large `write_file` content in a later change does not flood the terminal.
- Indicators are `ok`/`✓` and `error`/`✗`: the Unicode glyphs are used only when `console.encoding` is UTF-based, otherwise ASCII, so redirected output on a cp1252 Windows console never raises `UnicodeEncodeError`. Errors are red and carry the full message; successes show the summary only (the output goes to the model, not the terminal).
- All dynamic text (paths, error messages) is passed through `rich.markup.escape`, because file names and exception messages may contain `[...]`.
- **Who calls it:** the *caller* of `registry.execute` (the script now, the agent loop in RRG-63 later) wraps the call with `show_call`/`show_result`. The registry stays free of UI so it is reusable by subagents (RRG-73) and so RRG-72 hooks can sit around the same seam. **Alternative rejected:** `registry.execute(..., renderer=...)`, which couples the registry to Rich and hides ordering from the loop.
- Rich arrives here (scaffold-cli's design deferred it; RRG-61 uses `click.secho` for retry notices and plain `click.echo` for single mode). RRG-63 will unify the styling; this change only adds the tool-activity piece. Tests build `Console(file=io.StringIO(), width=100, force_terminal=False)` and assert on plain text.

### 9. Built-in registration and extension points
`builtin.py` has `register_builtin_tools(registry)` (explicit list: `ReadFileTool()`) and `create_default_registry(cwd: Path | None = None) -> ToolRegistry`. RRG-64..67 add one line each. There is no auto-discovery: an explicit list keeps order deterministic and avoids import scanning that RRG-75 will own for user tools. Extension points left for later changes (no code beyond these):
- **RRG-69 (approval):** reads `registry.get(name).requires_approval`; `validate_arguments` is public so the loop can validate before asking the user and then run. If a pre-run callback inside `execute` turns out to be cleaner, RRG-69 adds it.
- **RRG-72 (hooks):** `execute` is the single choke point for before/after tool events.
- **RRG-74/75:** register `Tool` instances whose `parameters` are foreign JSON Schemas; `register` already validates them and applies the name rule.

### 10. Developer runner (`scripts/run_tool.py`)
`python scripts/run_tool.py <tool_name> '<json args>'` (Click, argument default `{}`): parse JSON (invalid JSON or a non-object → `error: arguments are not valid JSON: <detail>` to stderr, exit code 2), build `create_default_registry(Path.cwd())`, call `ToolRenderer.show_call`, `registry.execute`, `show_result`, then print `result.output`. Exit code is 1 when the result is an error result and 0 otherwise, so the script is usable in shell checks. It is a script, not a `nar` subcommand, because the proposal adds no CLI surface.

### 11. Tests
`pytest` stays offline and uses `tmp_path`: registry (valid/invalid/duplicate names, unknown tool lists names, export order and shape), validation (missing, wrong type, unknown argument, minimum), exception-to-error-result conversion (a stub tool that raises), truncation (50,000 chars → `<= 30000` with marker), `read_file` (every spec scenario incl. a real PNG header, latin-1 file, 5,000-line file, relative path via a `ToolContext(cwd=tmp_path)`), renderer output on an in-memory console, and the script through `click.testing.CliRunner` (invalid JSON → non-zero). The manual checklist in `tasks.md` covers running the script in a real terminal.

## Risks / Trade-offs

- [`read_file` accepts any absolute path and `..`, so the model can read files outside the project (e.g. `~/.ssh`)] → The spec explicitly allows absolute paths and `requires_approval` is false; confinement or an approval rule belongs to RRG-69's autonomy levels. The risk is recorded here so that change picks it up.
- [`jsonschema` pulls in transitive dependencies (`attrs`, `referencing`, `rpds-py`)] → All publish Windows/macOS/Linux wheels for Python 3.11+; `pip install -e ".[dev]"` is still the whole setup. Pin only a lower bound.
- [`rich` and `jsonschema` edits to `pyproject.toml` can conflict with RRG-61's `anthropic`/`openai` additions] → Both changes append to the same `dependencies` list; resolve at merge, no semantic overlap.
- [Truncating by characters can cut a UTF-8 line mid-way or hide the end of a result] → The marker states how much was omitted; `read_file` puts its line-range statement first and the model can re-read with `offset`/`limit`.
- [Counting lines requires reading the whole file even for a 5-line request] → Bounded by the 10 MB cap; sequential read of at most 10 MB is fast enough for interactive use.
- [Catching every `Exception` in `execute` can hide programming bugs in tools] → The error result contains the exception type and message (spec), and the caller can log it; `NAR_DEBUG=1` behavior of the boundary is unchanged because no exception reaches it.
- [The shared dict from `Tool.parameters` could be mutated by a provider adapter] → `definition()` returns a deep copy.
- [`ok`/`error` ASCII fallback looks plainer than the Unicode glyphs] → Only used on non-UTF consoles; still satisfies the "visually distinct" requirement through color and the word `error`.

## Migration Plan

`pip install -e ".[dev]"` again to get `rich` and `jsonschema`. No config, CLI or data migration: nothing in the existing behavior changes until RRG-63 uses the registry. Rollback: revert the PR; `ToolError` and the new packages are additive.

## Open Questions

- Whether `ToolRegistry.execute` needs a pre-run callback for approval or hooks, or whether RRG-69/RRG-72 compose `get` + `validate_arguments` + `execute` in the loop. It does not affect any spec or task here and is decided in RRG-69's design.
- How RRG-74 shortens MCP tool names (`mcp__<server>__<tool>`) that exceed the 64-character name rule; it is owned by that change.
