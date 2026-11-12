# Tasks

> Review workload: about 6 lines of code and 50 of tests. One PR in `nar`, one commit.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-95 and verify the link opens `openspec/changes/fix-tool-bad-result/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-95 branch; verify `openspec validate fix-tool-bad-result --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-95 branch `rubenjustiniano96/rrg-95-fix-tool-bad-result-una-tool-que-no` from `main`, then make `ToolRegistry.execute` turn a value that is not a `ToolResult` into `ToolResult.error("<tool> returned <type name> instead of a ToolResult")` before `_finish`. Add tests in `tests/test_tools_registry.py` and `tests/test_tools_custom.py` for:
  - `None`, a `str` and a `dict` become error results that name the tool and the type
  - a real `ToolResult` passes through unchanged
  - a custom tool that returns `None` gives an error on that call and the next call still runs
  - the check also covers a tool registered like an MCP tool

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean.

## Manual verification

- [ ] 2.1 Walk through RRG-95's "done when" in a real terminal with a temp project and a temp `NAR_HOME`, recording what was observed:
  - put a custom tool in `.nar/tools/` whose `execute` returns `None`
  - run `nar`, ask for something that makes the model call it, and record the error shown for that call (`<tool> returned NoneType instead of a ToolResult`) and that there is no `unexpected failure`
  - ask another question and record that `nar` still answers
