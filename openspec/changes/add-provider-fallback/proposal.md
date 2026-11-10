# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-89 (sub-issue of RRG-61)
- **Video timestamps covered:** none (asked for by Nathalia on 2026-11-10)
- **Depends on:** RRG-86 `fix-model-switch-check` and RRG-87 `add-api-key-prompt` (both merged)
- **Course requirement covered:** selectable model provider

## Why

When the provider in use runs out of balance, goes over its quota or is down, `nar` prints the error and the request is lost. To go on, the user has to know about `/model`, switch, and type the request again. On OpenRouter the balance belongs to the account, not to a model, so another model of the same provider does not help.

## What Changes

- In interactive mode, when a request fails because the provider cannot answer, `nar` lists the other providers and lets the user pick one by number. "Cannot answer" means no balance, quota exceeded after the retries, or a server that is down or does not answer.
- Each option shows its default model and whether its key is set or will be asked for.
- The pick goes through the same path as `/model`: the key is asked for when missing (RRG-87), and the provider is checked (RRG-86). If that fails, the list comes back.
- After a switch, `nar` runs the same request again, with the whole conversation.
- Enter with no number stays as today.
- Nothing is offered for the user's own mistakes (a rejected key, an unknown model), in single mode, or without a terminal.

Out of scope: switching without asking (cost and privacy), and OpenRouter's own list of fallback models.

**"Done when" (RRG-89):** with a provider that has no balance or is down:
- a request shows the error and the list;
- picking another provider makes that provider answer the same request;
- Enter with no number leaves everything as today.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `llm-providers`: a new requirement that a provider error says whether the provider cannot answer. ADDED.
- `tui-commands`: a new requirement for offering another provider after such a failure in interactive mode. ADDED.

## Impact

- **Code (`nar` repo):**
  - `src/nar/errors.py` and `src/nar/providers/retry.py`, to mark the errors
  - `src/nar/ui/provider_menu.py` (new)
  - `src/nar/cli/fallback.py` (new)
  - `src/nar/cli/repl.py`
  - tests
- **Dependencies:** none new.
