# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-88 (sub-issue of RRG-78)
- **Video timestamps covered:** none (asked for by Nathalia on 2026-11-10, while trying `/model` with real keys)
- **Depends on:** RRG-78 `add-tui-commands` and RRG-86 `fix-model-switch-check` (both merged)
- **Course requirement covered:** selectable model provider

## Why

`/model <provider> <model>` needs the model's exact name, and `nar` cannot show the names: the user has to look them up on each provider's website. Since RRG-86, `nar` already asks the provider for its model list to check a switch, but it does not show it.

## What Changes

- `/models` lists the models of the provider in use and marks the one in use.
- `/models <provider>` lists another provider's models without switching.
- `/models <provider> <text>` keeps only the names that contain the text. This helps with OpenRouter, which has hundreds of models.
- It uses the same request as the check from RRG-86: no tokens, one attempt and a 10-second timeout.
- Errors are one line, and the session continues.

Out of scope: asking for a missing key from `/models` (`/model` does that), and showing prices or context sizes.

**"Done when" (RRG-88):**
- With DeepSeek in use, `/models` shows DeepSeek's models and marks the one in use.
- `/models anthropic` shows Anthropic's models without switching.
- `/models openrouter claude` shows only OpenRouter's models that contain "claude".

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `llm-providers`: a new requirement that a provider can list its models. ADDED.
- `tui-commands`: a new requirement for `/models`. ADDED.

## Impact

- **Code (`nar` repo):** `src/nar/providers/base.py`, `anthropic.py`, `openai_compat.py`, `src/nar/cli/commands.py`, `src/nar/cli/runner.py`, and tests.
- **Dependencies:** none new.
