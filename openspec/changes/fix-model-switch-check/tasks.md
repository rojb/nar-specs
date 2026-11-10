# Tasks

> Review workload: about 300 lines, mostly tests. It fits in one PR.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-86 and verify the link opens `openspec/changes/fix-model-switch-check/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-86 branch; verify `openspec validate fix-model-switch-check --strict` passes

## [nar]

- [ ] 1.1 Create Linear's RRG-86 branch from `main`, and add a test in `tests/test_cli.py` where `/model` gets a provider whose check fails; verify it fails before the fix because the switch happens
- [ ] 1.2 Add `LLMProvider.check()` and `unknown_model_error` in `src/nar/providers/base.py`, and implement `check` in the Anthropic, OpenAI-compatible, OpenRouter and Ollama adapters; verify with mocked HTTP: a rejected key, an unknown model with suggestions, Ollama unreachable with one attempt, an Ollama model not pulled, the `latest` tag, and a passing check that makes no model call
- [ ] 1.3 Call the check in `switch_model` before assigning the provider, and handle Ctrl+C in `/model`; verify the tests from 1.1 pass, Ctrl+C keeps the previous provider and the session, `ruff check .` and `ruff format --check .` are clean, and the only `pytest` failure, if any, is `test_keyboard_interrupt_kills_the_tree_and_propagates`, already accepted on Linux

## Manual verification

- [ ] 2.1 Walk through RRG-86's "done when" in a real terminal: start `nar` with a working provider, run `/model ollama` with Ollama stopped and record the error line, run `/model deepseek` with an invalid `DEEPSEEK_API_KEY` and record the error line, then ask a question and record that the previous provider answers. Then switch to a valid provider and record that it answers.
