# Tasks

> Review workload: under 100 lines, mostly tests. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-90 and verify the link opens `openspec/changes/fix-model-switch-context-window/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-90 branch; verify `openspec validate fix-model-switch-context-window --strict` passes _Evidence: rojb/nar-specs#28; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-90 branch from `main`, and add tests in `tests/test_cli.py` with fake providers. Verify they fail before the fix:
  - `/model ollama` then `/tokens` shows the new provider's window
  - with `context_window` set in `.nar/config.toml`, the window stays

  _Evidence: the window test failed on `main`. The test for a window the user set already passed there, because nothing changed the window at all; it guards that the fix keeps it._
- [x] 1.2 Add `context_window_set` to `NarConfig` and `fixed` to `ContextLimits`, and update the limit in `switch_model`. Verify:
  - the tests from 1.1 pass
  - a config test covers `context_window_set` for files, flags and defaults
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

  _Evidence: rojb/nar#41. `ruff` was clean, and `pytest` gave 1321 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-90's "done when" with the real DeepSeek key: start `nar --provider ollama`, run `/tokens`, run `/model deepseek`, run `/tokens` again, and record both lines

  _Evidence (Windows 11, 2026-11-10, real DeepSeek key, `nar --provider ollama`):_
  - _`/tokens` printed `context: 2,543 / 4,096 tokens (62%)`._
  - _`/model deepseek`, then `/tokens`, printed `context: 2,543 / 128,000 tokens (1%)`._
  - _`/model ollama` failed (Ollama is not installed), and the window stayed at `/ 128,000`._
  - _With `context_window = 32000` in `.nar/config.toml`, `/model deepseek` kept `/ 32,000 tokens`._
