# Design

## Context

The agent (RRG-63) builds one `ToolRegistry` per process (`create_default_registry(cwd)`), runs tools through it, and shows them with `ToolRenderer` (RRG-62), which already prints an optional `ToolResult.diff` below the success line (RRG-64). The system prompt comes from `build_system_prompt(cwd, instructions)` and is built once per process in `cli/runner.py:create_agent`. The user-level folder is `nar.config.paths.nar_home()` (`~/.nar` or `$NAR_HOME`). Motivation: see `proposal.md`. Behavior: see `specs/todos-tool`, `specs/memory-tool`, `specs/tool-framework`.

Constraints:
- Windows and Linux; the memory file must survive a crash mid-write.
- Tests must not touch the real home folder (they set `NAR_HOME`).

## Goals / Non-Goals

**Goals:**
- A checklist the user sees on every change, without a new UI component.
- A memory small enough to always fit in the system prompt.
- No change to the agent loop or the registry API.

**Non-Goals:**
- Persisting todos (RRG-76), memory search or embeddings, separate project and user memories.

## Decisions

### 1. Layout
```
src/nar/tools/
├── base.py        MODIFIED  ToolResult.display (optional), ToolResult.ok(display=...)
├── todos.py       NEW       TodosTool, TodoItem
├── memory.py      NEW       MemoryTool, MemoryStore, MemoryFileError, memory_path()
└── builtin.py     MODIFIED  registers TodosTool, MemoryTool after WebFetchTool
src/nar/ui/tool_view.py      MODIFIED  ToolRenderer prints result.display
src/nar/agent/prompt.py      MODIFIED  guidelines + memory section; build_system_prompt(memory=...)
src/nar/cli/runner.py        MODIFIED  loads memory for the system prompt
```

### 2. `ToolResult.display`
A new last field `display: str | None = None`, and `ToolResult.ok(..., display=None)`. `ToolRegistry._finish` already uses `dataclasses.replace`, so truncation keeps it. `ToolRenderer.show_result` prints it after the diff (if any), one `rich.text.Text` per line so markup is never interpreted; lines starting with `[x]` are dim, lines starting with `[>]` bold, the rest unstyled. The model never sees `display`; tools put what the model needs in `output`.
- **Why a generic field instead of special-casing `todos` in the view:** RRG-62 keeps tool-specific knowledge out of the UI; the same field can serve later tools (for example, a shell command's exit code banner in RRG-65). **Alternative rejected:** a multi-line `summary` (the contract says one line).

### 3. `todos`
Schema: `todos` (array, required, `maxItems: 50`) of objects `{content: string minLength 1, status: enum pending|in_progress|completed}`, `required` both, `additionalProperties: false`. The registry's JSON Schema validation produces the field-level errors (spec: invalid status). The tool checks the single-in-progress rule and keeps `self.items` only on success.
- Output for the model: the checklist, then `N completed, M remaining`. `display` = the checklist. `summary` = `N tasks (C completed)`. Empty list → output `task list is empty`, summary `cleared task list`, no display.
- State lives in the tool instance; one registry per process → per session (spec: list scope).

### 4. `memory`
File: `memory_path() = nar_home() / "memory.json"`, evaluated at each call so `NAR_HOME` set by tests is honored. Format:
```json
{"version": 1, "entries": {"language": "Answer in Spanish"}}
```
`MemoryStore(path)`: `load() -> dict[str, str]` (missing file → `{}`; invalid JSON, wrong shape or non-string values → `MemoryFileError(NarError)` naming the file); `save(key, value)` and `delete(key)` load, change and write. Writes go to a temporary file in the same folder and then `os.replace` (atomic on Windows and POSIX); keys are written sorted, `ensure_ascii=False`, indented. The parent folder is created on the first save.

Schema: `action` (enum `save|delete|list`, required), `key`, `value` (strings), `additionalProperties: false`. The tool checks, with explicit messages: required arguments per action; key `^[A-Za-z0-9_.-]{1,64}$`; value 1–500 characters (after stripping); at most 50 entries (replacing an existing key is allowed when full). Messages: `saved language: Answer in Spanish`, `deleted language`, `list` → `key: value` lines or `(memory is empty)`; unknown key → `no memory entry 'x'; stored keys: a, b`.
- **Why limits:** every entry is sent in every system prompt; 50 × ~560 characters stays under ~7,000 tokens worst case and normally far less.
- **Why no approval:** the issue's "done when" runs at the default `ask` level, and the file only holds short text the user can read and edit. Trade-off: a prompt-injected page could make the model save an instruction that persists; the user sees every `memory` call in the tool activity, and RRG-69 can flip `requires_approval`.

### 5. System prompt
`build_system_prompt(cwd, instructions=None, memory=None, *, platform_name=None, today=None)`. Two guideline lines are added:
- `For tasks with several steps, first write a plan with the todos tool, keep exactly one task in progress, and update the list as you finish each step.`
- `When the user states a lasting preference or an important fact about themselves or the project, save it with the memory tool (a short key and a one-sentence value). Follow the facts listed under Memory.`
When `memory` has entries, a section `# Memory (saved in earlier sessions)` with `- key: value` lines follows the environment section and precedes the project instructions. `create_agent` loads `MemoryStore(memory_path()).load()`; a `MemoryFileError` becomes a warning through `AgentView.warn` and the session starts without memory (spec: unreadable file).

### 6. Tests
`tests/test_tools_todos.py`, `tests/test_tools_memory.py` (`NAR_HOME` on `tmp_path`; atomic write checked by making `os.replace` fail and asserting the old file is intact), `tests/test_tool_view.py` (display lines literal and styled), `tests/test_system_prompt.py` (memory section present/absent, guideline lines), CLI test with the fake provider asserting a saved entry appears in the next run's system prompt and that a corrupt file gives a warning.

## Risks / Trade-offs

- [Models that ignore the planning guideline] → Harmless; the tool is optional. The checklist only appears when used.
- [Memory poisoning by prompt injection] → Decision 4 (visible calls, tiny limits, approval flag available to RRG-69).
- [Two `nar` processes saving at once] → Last writer wins for the whole file; acceptable for a single-user CLI. Atomic replace prevents corruption.
- [Archive order] → This change modifies `tool-framework` only; it does not depend on `add-agent-loop` being archived.

## Migration Plan

No migration. The memory file appears on the first save. Rollback: revert the PR; an existing `memory.json` is simply ignored.
