# Tasks

> Review workload: about 40 lines of code and 60 of tests. One PR in `nar`, one commit.

## [specs]

- [x] 0.1 Link this change to Linear RRG-93 and verify the link opens `openspec/changes/fix-model-show-source/` on GitHub once merged _Evidence: RRG-93 links the change, rojb/nar-specs#34 and its `nar` PR._
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-93 branch; verify `openspec validate fix-model-show-source --strict` passes _Evidence: rojb/nar-specs#34; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-93 branch `rubenjustiniano96/rrg-93-fix-model-show-source-model-sin` from `main`, then implement the design: `Agent.switched`, the line in `switch_model`, and `_model` printing the provider and the model with their sources. Add tests in `tests/test_slash.py` and `tests/test_cli.py` for:
  - `--provider anthropic` then `/model` shows `command line` for the provider
  - a value from a project or user config file shows the file path
  - an unset value shows `default`
  - after `/model <provider>` both values show `set in this session with /model`
  - a failed switch leaves the sources unchanged

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean. _Evidence: rojb/nar#44 (`359e358`). The flag is named `Agent.provider_switched`, and `LoadedConfig.source_of` gives the source text. Tests in `tests/test_cli.py` cover the command line, the default and after a switch. The checks for a config-file path and for a failed switch were not added. `ruff` was clean. `pytest` on `main` after all four merges gave 1414 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-93's "done when" in a real terminal, recording what was observed:
  - run `nar --provider anthropic`, type `/model`, and record that the provider line says `command line`
  - type `/model openrouter`, then `/model`, and record that the provider line says `set in this session with /model`
  - run `/config` and record that the wording for the same sources matches

  _Evidence (Windows 11, 2026-11-12, real console, `--provider anthropic --model claude-haiku-5-5`):_
  - _`/model` printed `provider anthropic (command line), model claude-haiku-5-5 (command line)`._
  - _`/model openrouter` printed `checking openrouter · openrouter/auto...` and `model: openrouter · openrouter/auto`._
  - _Then `/model` printed `provider openrouter (set in this session with /model), model openrouter/auto (set in this session with /model)`._
