# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-86 (sub-issue of RRG-78)
- **Video timestamps covered:** none (found while writing the test plan on 2026-11-10)
- **Depends on:** RRG-78 `add-tui-commands`, RRG-61 `add-llm-providers` and RRG-81 `fix-compaction-model-switch` (all merged)
- **Course requirement covered:** selectable model provider, as a fix

## Why

`/model` only checks that the API key variable exists. It does not check that the provider answers, that the key is valid or that the model exists, and it still reports `model: <provider> · <model>`. The error shows up on the next request instead, and on every request after it, until the user switches again. With Ollama not installed, `/model ollama` reported success, and then each request took about 8 seconds over 4 attempts before `cannot reach Ollama at http://localhost:11434`.

## What Changes

- Every provider can check, without a model call, that it answers, that its key is accepted and that its model exists.
- `/model` runs that check before switching. When it fails, it prints a one-line error that names the provider still in use, suggests close model names when there are any, and changes nothing.
- Ctrl+C during the check cancels the switch and keeps the session.

Out of scope: checking at startup, DeepSeek's balance, and asking for a missing key in the terminal.

**"Done when" (RRG-86):** `/model ollama` with Ollama stopped and `/model <provider>` with an invalid key print a one-line error, and the next request is still answered by the previous provider.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `llm-providers`: a new requirement that a provider can check its access without a model call. ADDED.
- `tui-commands`: a new requirement that `/model` checks the new provider before switching. ADDED, so the existing `/model` requirement keeps its text.

## Impact

- **Code (`nar` repo):** `src/nar/providers/base.py`, `anthropic.py`, `openai_compat.py`, `src/nar/cli/runner.py` (`switch_model`), `src/nar/cli/commands.py` (`/model`), and tests in `tests/test_provider_anthropic.py`, `tests/test_provider_openai_compat.py` and `tests/test_cli.py`.
- **Dependencies:** none new.
- **Cost:** one extra HTTP request per `/model`, which uses no tokens.
