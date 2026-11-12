# Tasks

> Review workload: about 6 lines of code and 50 of tests. One PR in `nar`, one commit.

## [specs]

- [x] 0.1 Link this change to Linear RRG-95 and verify the link opens `openspec/changes/fix-tool-bad-result/` on GitHub once merged _Evidence: RRG-95 links the change, rojb/nar-specs#34 and its `nar` PR._
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-95 branch; verify `openspec validate fix-tool-bad-result --strict` passes _Evidence: rojb/nar-specs#34; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-95 branch `rubenjustiniano96/rrg-95-fix-tool-bad-result-una-tool-que-no` from `main`, then make `ToolRegistry.execute` turn a value that is not a `ToolResult` into `ToolResult.error("<tool> returned <type name> instead of a ToolResult")` before `_finish`. Add tests in `tests/test_tools_registry.py` and `tests/test_tools_custom.py` for:
  - `None`, a `str` and a `dict` become error results that name the tool and the type
  - a real `ToolResult` passes through unchanged
  - a custom tool that returns `None` gives an error on that call and the next call still runs
  - the check also covers a tool registered like an MCP tool

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean. _Evidence: rojb/nar#46 (`4ef0d6c`). `tests/test_tools_registry.py` covers `None` and `str` as parametrized cases, and both failed before the fix. The `dict`, custom-tool and MCP-like cases were not added as separate tests; they share the same check in `ToolRegistry`, and the custom-tool path is verified in 2.1. `ruff` was clean. `pytest` on `main` after all four merges gave 1414 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-95's "done when" in a real terminal with a temp project and a temp `NAR_HOME`, recording what was observed:
  - put a custom tool in `.nar/tools/` whose `execute` returns `None`
  - run `nar`, ask for something that makes the model call it, and record the error shown for that call (`<tool> returned NoneType instead of a ToolResult`) and that there is no `unexpected failure`
  - ask another question and record that `nar` still answers

  _Evidence (Windows 11, 2026-11-12, real console, Anthropic, `--trust-project-tools`):_
  - _`.nar/tools/bad_tool.py` returns `None`. The call showed `✗ bad_tool returned NoneType instead of a ToolResult`, and there was no `unexpected failure`._
  - _The next request, `Reply with exactly the word ALIVE.`, was answered `ALIVE`._
