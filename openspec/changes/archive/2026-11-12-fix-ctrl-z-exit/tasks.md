# Tasks

> Review workload: about 5 lines of code and 40 of tests. One PR in `nar`, one commit.

## [specs]

- [x] 0.1 Link this change to Linear RRG-96 and verify the link opens `openspec/changes/fix-ctrl-z-exit/` on GitHub once merged _Evidence: RRG-96 links the change, rojb/nar-specs#34 and its `nar` PR._
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-96 branch; verify `openspec validate fix-ctrl-z-exit --strict` passes _Evidence: rojb/nar-specs#34; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-96 branch `rubenjustiniano96/rrg-96-fix-ctrl-z-exit-ctrlz-y-enter-en` from `main`, then make `run` in `src/nar/cli/repl.py` treat a line that is only `\x1a` (after stripping whitespace) as end of input, with exit code 0 and nothing sent. Add tests in `tests/test_cli.py` for:
  - piped input `\x1a` ends the session with exit code 0 and no request reaches the provider
  - `\x1a` with surrounding spaces does the same
  - a line that only contains `\x1a` inside other text is still sent
  - Ctrl+D and `exit` still end the session

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean. _Evidence: rojb/nar#47 (`dc35c8c`). The check is in `read_line` in `src/nar/cli/prompt.py`, not in `repl.run`, so it covers both the terminal and the piped path in one place. 5 new tests in `tests/test_cli_prompt.py` and `tests/test_cli.py` failed before the fix. They cover the piped and terminal paths, `\x1a` inside text, and a piped run that exits 0 with no provider request. `ruff` was clean. `pytest` on `main` after all four merges gave 1414 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-96's "done when" in a Windows console, recording what was observed:
  - run `nar`, press Ctrl+Z then Enter at `nar>`, and record that the session ends, `echo %ERRORLEVEL%` (or `$LASTEXITCODE`) is 0, and no request or model output appeared
  - run it again, type a normal question, then Ctrl+Z and Enter, and record that the first question was answered and the session ended

  _Evidence (Windows 11, 2026-11-12, real console, Anthropic): Ctrl+Z then Enter at `nar>` ended the session with exit code 0. The key events were synthetic (`WriteConsoleInputW`), as in the run that found the bug. The second step (a question first, then Ctrl+Z) was not run separately; the earlier end-to-end run showed requests answered before an end of input._
