# Tasks

## [specs]

- [x] 0.1 Link this change to Linear RRG-68 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/add-planning-memory-tools/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-68 branch with proposal, specs, design and tasks; verify `openspec validate add-planning-memory-tools --strict` passes

## [nar]

## 1. Branch

- [ ] 1.1 In `nar/`, update `main` and create the branch `nathaliascode/rrg-68-add-planning-memory-tools-todos-planificacion-y-memoria`; verify `git status` shows the new branch and a clean tree

## 2. Extra display text for tool results

- [ ] 2.1 Add `display` to `ToolResult` and `ToolResult.ok()` and print it in `ToolRenderer.show_result` (design.md §2), extending `tests/test_tools_base.py`, `tests/test_tools_registry.py` and `tests/test_tool_view.py`; verify `display` defaults to `None`, survives registry truncation, is printed below the success line (after a diff when both exist), `[red]` inside it prints literally, `[x]` lines are dim and `[>]` lines bold on a color console, and error results print no display

## 3. todos tool

- [ ] 3.1 Create `src/nar/tools/todos.py` (`TodosTool`, design.md §3) with `tests/test_tools_todos.py`; verify every scenario of `specs/todos-tool/spec.md`: create a plan, update progress, invalid status rejected by the registry naming `status`, two in-progress items rejected with the previous list kept, checklist markers in `display` and in `output` with the completed/remaining counts, empty array clears, the list persists on the same instance; plus `TodosTool().requires_approval is False`

## 4. memory tool

- [ ] 4.1 Create `MemoryStore`, `MemoryFileError` and `memory_path()` in `src/nar/tools/memory.py` (design.md §4) with `tests/test_tools_memory.py`; verify load of a missing file is `{}` and creates nothing, save/delete round-trip through a new store instance, invalid JSON / wrong shape / non-string values raise `MemoryFileError` naming the file, `NAR_HOME` decides the path, the file is written sorted with `ensure_ascii=False`, and a failing `os.replace` leaves the previous file intact
- [ ] 4.2 Add `MemoryTool` to `src/nar/tools/memory.py` and extend `tests/test_tools_memory.py`; verify every scenario of `specs/memory-tool/spec.md` for the tool: save, replace, delete, missing `value`/`key`, unknown key listing stored keys, list order, empty memory without creating a file, a 2,000-character value refused with the 500 limit, a 51st key refused while replacing an existing key works at 50, an invalid key refused, a corrupt file left unchanged; plus `MemoryTool().requires_approval is False`

## 5. Registration, system prompt and CLI

- [ ] 5.1 Register `TodosTool` and `MemoryTool` after `WebFetchTool` in `src/nar/tools/builtin.py` and update `tests/test_tools_registry.py`; verify the default registry names end with `"web_fetch", "todos", "memory"`
- [ ] 5.2 Add the two guideline lines and the `memory` parameter/section to `build_system_prompt` (design.md §5), extending `tests/test_system_prompt.py`; verify the section lists `- key: value` lines in order between the environment and the project instructions, and is absent without entries
- [ ] 5.3 Load the memory in `create_agent` (`src/nar/cli/runner.py`) with a warning on `MemoryFileError`; extend `tests/test_cli.py` with the fake provider; verify an entry saved through the tool in one run appears in the next run's system prompt, and a corrupt `memory.json` gives a warning on stderr while the request still runs
- [ ] 5.4 Add the two tools to the README's built-in tools table and a short section (planning checklist, memory file location and limits, how to inspect or delete it) with `scripts/run_tool.py` examples; verify the documented commands run as written with `NAR_HOME` pointing at a temp folder

## 6. Delivery

- [ ] 6.1 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR in `nar` referencing RRG-68

## 7. Manual verification (RRG-68 "done when")

- [ ] 7.1 Walk through RRG-68's "done when" ("in a new session the agent remembers a preference saved in an earlier session") at the `nar/` repo root with `NAR_HOME` pointing at a temp folder, and record what was observed for each step:
  - `nar --provider openrouter "recuerda que prefiero respuestas cortas y con ejemplos en Python"` → stderr shows a `memory` call with `action="save"`; `$NAR_HOME/memory.json` contains the entry; exit 0
  - New process: `nar --provider openrouter "¿qué preferencias mías recuerdas?"` → no tool call needed; the answer mentions the saved preference; exit 0
  - Planning: `nar --provider openrouter "revisa src/nar/tools/todos.py y src/nar/tools/memory.py y dame un resumen de cada uno"` → stderr shows a `todos` call with the checklist below it, and later updates with `[x]` items
  - Same memory check with `--provider ollama --model <local model>` (and `anthropic` when a key is available)
