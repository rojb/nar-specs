# Tasks

> Review workload: about 5 lines of code and 40 of tests. One PR in `nar`, one commit.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-96 and verify the link opens `openspec/changes/fix-ctrl-z-exit/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-96 branch; verify `openspec validate fix-ctrl-z-exit --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-96 branch `rubenjustiniano96/rrg-96-fix-ctrl-z-exit-ctrlz-y-enter-en` from `main`, then make `run` in `src/nar/cli/repl.py` treat a line that is only `\x1a` (after stripping whitespace) as end of input, with exit code 0 and nothing sent. Add tests in `tests/test_cli.py` for:
  - piped input `\x1a` ends the session with exit code 0 and no request reaches the provider
  - `\x1a` with surrounding spaces does the same
  - a line that only contains `\x1a` inside other text is still sent
  - Ctrl+D and `exit` still end the session

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean.

## Manual verification

- [ ] 2.1 Walk through RRG-96's "done when" in a Windows console, recording what was observed:
  - run `nar`, press Ctrl+Z then Enter at `nar>`, and record that the session ends, `echo %ERRORLEVEL%` (or `$LASTEXITCODE`) is 0, and no request or model output appeared
  - run it again, type a normal question, then Ctrl+Z and Enter, and record that the first question was answered and the session ended
