# Tasks

> Review workload: about 5 lines of code and 40 of tests. One PR in `nar`, one commit.

## [specs]

- [x] 0.1 Link this change to Linear RRG-94 and verify the link opens `openspec/changes/fix-hook-block-warning/` on GitHub once merged _Evidence: RRG-94 links the change, rojb/nar-specs#34 and its `nar` PR._
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-94 branch; verify `openspec validate fix-hook-block-warning --strict` passes _Evidence: rojb/nar-specs#34; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-94 branch `rubenjustiniano96/rrg-94-fix-hook-block-warning-un-hook-que` from `main`, then make `HookRunner._report` return early (after showing the hook's output) for a `before_tool` outcome that blocks. Extend `tests/test_hooks_runner.py` to check that:
  - a `before_tool` hook that exits 2 blocks the call, returns the refused result with the hook's text, and writes no warning
  - exit 1 and exit 3 still warn with `did not block the call`
  - a missing command and a timeout still warn
  - an `after_tool` hook that exits 2 still warns

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean. _Evidence: rojb/nar#45 (`0045abd`), `tests/test_hooks_runner.py`. The exit-2 test failed before the fix. Exit 2 from `after_tool` still warns (new test). Exit 1, 3 and 127 are covered by the existing tests. `ruff` was clean. `pytest` on `main` after all four merges gave 1414 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-94's "done when" in a real terminal with a temp project and a temp `NAR_HOME`, recording what was observed:
  - set `before_tool = ["Write-Output 'blocked by policy'; exit 2"]`, ask for an edit, and record that the file stayed untouched, that the refusal notice and the hook's sentence appear, and that no `warning: hook before_tool failed` line appears
  - change it to `exit 1`, ask for the edit again, and record that the edit ran and the warning says the call was not blocked

  _Evidence (Windows 11, 2026-11-12, real console, Anthropic):_
  - _With `before_tool = ['Write-Output "BLOCKED by policy"; exit 2']`, the call printed `✗ the before_tool hook refused this call: BLOCKED by policy`, with no `warning: hook` line._
  - _With `before_tool = ['exit 1']`, the output was `warning: hook before_tool failed with exit code 1 and did not block the call: exit 1`, then `✓ listed 2 entries`._
