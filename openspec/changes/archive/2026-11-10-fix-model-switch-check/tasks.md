# Tasks

> Review workload: about 300 lines, mostly tests. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-86 and verify the link opens `openspec/changes/fix-model-switch-check/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-86 branch; verify `openspec validate fix-model-switch-check --strict` passes _Evidence: rojb/nar-specs#21; the validation passed._

## [nar]

- [x] 1.1 Create Linear's RRG-86 branch from `main`, and add a test in `tests/test_cli.py` where `/model` gets a provider whose check fails; verify it fails before the fix because the switch happens _Evidence: two of the three new tests failed on `main` (the switch happened without a check); the Ctrl+C test passed there because nothing was checked._
- [x] 1.2 Add `LLMProvider.check()` and `unknown_model_error` in `src/nar/providers/base.py`, and implement `check` in the Anthropic, OpenAI-compatible, OpenRouter and Ollama adapters; verify with mocked HTTP: a rejected key, an unknown model with suggestions, Ollama unreachable with one attempt, an Ollama model not pulled, the `latest` tag, and a passing check that makes no model call _Evidence: 13 new tests in `tests/test_provider_anthropic.py` and `tests/test_provider_openai_compat.py`, plus a timeout case. The suggestion cutoff went from 0.6 to 0.8 (see design)._
- [x] 1.3 Call the check in `switch_model` before assigning the provider, and handle Ctrl+C in `/model`; verify the tests from 1.1 pass, Ctrl+C keeps the previous provider and the session, `ruff check .` and `ruff format --check .` are clean, and the only `pytest` failure, if any, is `test_keyboard_interrupt_kills_the_tree_and_propagates`, already accepted on Linux _Evidence: rojb/nar#37. `ruff` was clean, and `pytest` gave 1258 passed and 3 skipped on Windows, with no failures._

## Manual verification

- [x] 2.1 Walk through RRG-86's "done when" in a real terminal: start `nar` with a working provider, run `/model ollama` with Ollama stopped and record the error line, run `/model deepseek` with an invalid `DEEPSEEK_API_KEY` and record the error line, then ask a question and record that the previous provider answers. Then switch to a valid provider and record that it answers.

  _Evidence 1 (real servers, fake keys, no Ollama installed, Windows 11, 2026-11-10): starting on DeepSeek, these four commands each printed one error line ending with `still using deepseek · deepseek-flash`, and `/model` still showed DeepSeek. The five commands took 12 seconds in total:_
  - _`/model ollama`: `cannot reach Ollama at http://localhost:11434; is it running? (start it with 'ollama serve')`_
  - _`/model anthropic`: `anthropic rejected the API key; check ANTHROPIC_API_KEY`_
  - _`/model openrouter`: `openrouter rejected the API key; check OPENROUTER_API_KEY`_
  - _`/model deepseek deepseek-flsh`: `deepseek rejected the API key; check DEEPSEEK_API_KEY`_

  _Evidence 2 (a real Windows console with keys typed through `WriteConsoleInput`, and a simulated Anthropic server reached with `ANTHROPIC_BASE_URL`; `pruebas/demo-rrg86-87.py`, run while Nathalia watched, 18 of 18 checks):_
  - _After a question answered by `anthropic`, `/model ollama` printed `checking ollama · qwen2.5-coder...` and an error ending with `still using anthropic · claude-sonnet-5-5`._
  - _The session went on with the same provider._

  _Not run on 2026-11-10 at archive time: a question answered by a real provider after the failed switch, because the team had no real key yet._

  _Evidence 3, added after the archive (real DeepSeek and Anthropic keys, 2026-11-10, during RRG-88). Starting on DeepSeek, after a question DeepSeek answered:_
  - _`/model anthropic claude-sonet-5-5` printed `model 'claude-sonet-5-5' was not found by anthropic; did you mean claude-sonnet-5-5, claude-sonnet-5, claude-sonnet-4-6?; still using deepseek · deepseek-flash`._
  - _`/model ollama` printed `cannot reach Ollama ...; still using deepseek · deepseek-flash`._
  - _`/model anthropic` printed `model: anthropic · claude-sonnet-5-5`. The next question was answered by Claude, which referred to DeepSeek's earlier answer, so the conversation continued._
  - _`/model deepseek` switched back._
