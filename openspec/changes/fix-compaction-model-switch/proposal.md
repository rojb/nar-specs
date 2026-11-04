# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-81
- **Video timestamps covered:** none (bug found while reading the code for RRG-73)
- **Depends on:** RRG-70 `add-context-compaction` and RRG-78 `add-tui-commands` (both merged)
- **Course requirement covered:** context management and selectable model provider, both only as a fix

## Why

After `/model`, the agent answers with the new provider, but compaction still asks the previous one for its summary. `Compactor` keeps its own provider, and `switch_model` only changes `agent.provider`. So the conversation can be sent to a provider the user has just left, which is also the wrong one to depend on if its key or balance is the reason for switching.

## What Changes

- `/model` also updates the provider that automatic compaction and `/compact` use.

Out of scope: subagents, which already read the current provider on each call.

**"Done when" (RRG-81):** after `/model`, `/compact` asks the new provider for the summary.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `context-compaction`: a new requirement that compaction uses the provider currently selected. ADDED, so the change does not depend on the archive of `add-context-compaction`.

## Impact

- **Code (`nar` repo):** `src/nar/cli/runner.py` (`switch_model`), `tests/test_cli.py`.
- **Dependencies:** none new.
