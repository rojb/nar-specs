# Tasks

> Review workload: under 30 lines. It fits in one PR.

## [specs]

- [x] 0.1 Link this change to Linear RRG-81 and verify the link opens `openspec/changes/fix-compaction-model-switch/` on GitHub once merged
- [x] 0.2 Open a PR in `nar-specs` from Linear's RRG-81 branch; verify `openspec validate fix-compaction-model-switch --strict` passes

## [nar]

- [x] 1.1 Create Linear's RRG-81 branch from `main`, and add a test in `tests/test_cli.py` that runs `/model` and then `/compact`; verify it fails before the fix _Evidence: it failed on `main`, with fake providers._
- [x] 1.2 Update the compactor's provider in `switch_model`; verify the test passes, `ruff check .` and `ruff format --check .` are clean, and the only `pytest` failure is `test_keyboard_interrupt_kills_the_tree_and_propagates`, already accepted on Linux

## Manual verification

- [ ] 2.1 Walk through RRG-81's "done when" with OpenRouter: make a request, run `/model` to a model name that does not exist, run `/compact`, and record that the error names the new model, which shows the summary went to the new provider. Ollama is not run, at Nathalia's request (2026-11-03). Anthropic is pending until the team has an `ANTHROPIC_API_KEY`

  _Evidence (OpenRouter, a real terminal through a pty, 2026-11-04): after a request, `/model openrouter/no-existe/modelo-falso` printed `model: openrouter · no-existe/modelo-falso`, and `/compact` printed `error: openrouter rejected the request (400): no-existe/modelo-falso is not a valid model ID`. The summary went to the new model, and the session continued to `exit` with code 0. `pytest`: 1242 passed; the only failure is the known Linux one._

  _Not run with Ollama, at Nathalia's request (2026-11-03). Still open: Anthropic._

> **Closed on 2026-11-04 by decision of Nathalia.** The Anthropic check was not run, because the team has no `ANTHROPIC_API_KEY`, and the Ollama check was not run, at her request. Unchecked tasks above were left unchecked on purpose. Ruben was told in Linear RRG-61.
