# Tasks

> Review workload: under 100 lines, mostly tests. It fits in one PR.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-90 and verify the link opens `openspec/changes/fix-model-switch-context-window/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-90 branch; verify `openspec validate fix-model-switch-context-window --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-90 branch from `main`, and add tests in `tests/test_cli.py` with fake providers. Verify they fail before the fix:
  - `/model ollama` then `/tokens` shows the new provider's window
  - with `context_window` set in `.nar/config.toml`, the window stays
- [ ] 1.2 Add `context_window_set` to `NarConfig` and `fixed` to `ContextLimits`, and update the limit in `switch_model`. Verify:
  - the tests from 1.1 pass
  - a config test covers `context_window_set` for files, flags and defaults
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

## Manual verification

- [ ] 2.1 Walk through RRG-90's "done when" with the real DeepSeek key: start `nar --provider ollama`, run `/tokens`, run `/model deepseek`, run `/tokens` again, and record both lines
