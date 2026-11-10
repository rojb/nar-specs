# Tasks

> Review workload: about 200 lines, mostly tests. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-88 and verify the link opens `openspec/changes/add-models-command/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-88 branch; verify `openspec validate add-models-command --strict` passes _Evidence: rojb/nar-specs#24; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-88 branch from `main`, and add `list_models()` to `LLMProvider` and the adapters, with `check` reusing it. Verify with mocked HTTP:
  - the names are returned, and no model call is made
  - a rejected key names the variable
  - Ollama unreachable fails once
  - the `check` tests from RRG-86 still pass

  _Evidence: four new tests in `tests/test_provider_openai_compat.py` and `tests/test_provider_anthropic.py`. The RRG-86 `check` tests pass unchanged._
- [x] 1.2 Add `/models` in `src/nar/cli/commands.py` and `list_models` in `src/nar/cli/runner.py`. Verify with fake providers in `tests/test_cli.py`:
  - the provider in use is listed and its model is marked
  - another provider is listed without switching
  - the filter works, including a filter that matches nothing
  - an unknown provider and a missing key print one line
  - Ctrl+C cancels only the listing
  - `ruff check .`, `ruff format --check .` and `pytest` are clean

  _Evidence: rojb/nar#39. `ruff` was clean, and `pytest` gave 1297 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-88's "done when" with the real keys:
  - with DeepSeek in use, run `/models`
  - run `/models anthropic` and `/models openrouter claude`
  - after each one, run `/model` and record that the provider did not change

  _Evidence (real DeepSeek and Anthropic keys, read from the Windows user environment and never printed, 2026-11-10). Starting on DeepSeek:_
  - _`/models` printed `deepseek: 2 models` and marked `deepseek-flash  ← in use`._
  - _`/models deepseek flash` printed `deepseek: 1 of 2 models match 'flash'`._
  - _`/models anthropic` printed `anthropic: 14 models`._
  - _`/models anthropic opus` printed `anthropic: 6 of 14 models match 'opus'`._
  - _At the end, `/model` still showed `provider deepseek, model deepseek-flash`._

  _`/models openrouter claude` printed `error: OPENROUTER_API_KEY is not set`, because the team has no OpenRouter key. The filter was checked with `opus` and `flash` instead._
