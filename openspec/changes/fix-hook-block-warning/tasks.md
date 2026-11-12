# Tasks

> Review workload: about 5 lines of code and 40 of tests. One PR in `nar`, one commit.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-94 and verify the link opens `openspec/changes/fix-hook-block-warning/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-94 branch; verify `openspec validate fix-hook-block-warning --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-94 branch `rubenjustiniano96/rrg-94-fix-hook-block-warning-un-hook-que` from `main`, then make `HookRunner._report` return early (after showing the hook's output) for a `before_tool` outcome that blocks. Extend `tests/test_hooks_runner.py` to check that:
  - a `before_tool` hook that exits 2 blocks the call, returns the refused result with the hook's text, and writes no warning
  - exit 1 and exit 3 still warn with `did not block the call`
  - a missing command and a timeout still warn
  - an `after_tool` hook that exits 2 still warns

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean.

## Manual verification

- [ ] 2.1 Walk through RRG-94's "done when" in a real terminal with a temp project and a temp `NAR_HOME`, recording what was observed:
  - set `before_tool = ["Write-Output 'blocked by policy'; exit 2"]`, ask for an edit, and record that the file stayed untouched, that the refusal notice and the hook's sentence appear, and that no `warning: hook before_tool failed` line appears
  - change it to `exit 1`, ask for the edit again, and record that the edit ran and the warning says the call was not blocked
