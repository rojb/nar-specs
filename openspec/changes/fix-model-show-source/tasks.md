# Tasks

> Review workload: about 40 lines of code and 60 of tests. One PR in `nar`, one commit.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-93 and verify the link opens `openspec/changes/fix-model-show-source/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-93 branch; verify `openspec validate fix-model-show-source --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-93 branch `rubenjustiniano96/rrg-93-fix-model-show-source-model-sin` from `main`, then implement the design: `Agent.switched`, the line in `switch_model`, and `_model` printing the provider and the model with their sources. Add tests in `tests/test_slash.py` and `tests/test_cli.py` for:
  - `--provider anthropic` then `/model` shows `command line` for the provider
  - a value from a project or user config file shows the file path
  - an unset value shows `default`
  - after `/model <provider>` both lines show `this session (/model)`
  - a failed switch leaves the sources unchanged

  Verify `ruff check .`, `ruff format --check .` and `pytest` are clean.

## Manual verification

- [ ] 2.1 Walk through RRG-93's "done when" in a real terminal, recording what was observed:
  - run `nar --provider anthropic`, type `/model`, and record that the provider line says `command line`
  - type `/model openrouter`, then `/model`, and record that the provider line says `this session (/model)`
  - run `/config` and record that the wording for the same sources matches
