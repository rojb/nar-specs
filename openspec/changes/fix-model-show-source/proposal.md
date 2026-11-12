# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-93
- **Video timestamps covered:** none (found by the real-API end-to-end run on 2026-11-12)
- **Depends on:** none
- **Course requirement covered:** selectable model provider (the `/model` command)

## Why

The `tui-commands` spec says `/model` with no argument shows the provider and model in use, with the source they came from. The real-API run of 2026-11-12 showed that it prints only `provider anthropic, model claude-haiku-5-5`. The user cannot tell whether that came from the default, a config file, a flag or an earlier `/model`.

## What Changes

- `/model` with no argument prints the provider and the model on their own lines, each with its source.
- The source uses the same wording as `/config`: `default`, the config file path, or `command line`.
- After a successful `/model <provider>` switch, `/model` shows `set in this session with /model` as the source of what was switched.
- A switch that fails changes nothing, so the sources stay as they were.

Out of scope: changing `/config`, changing how a switch is checked or reported, and saving the choice to a config file.

**"Done when" (RRG-93):**
- `nar --provider anthropic` then `/model` shows that the provider came from the command line;
- after `/model openrouter`, `/model` shows that the provider came from this session.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

None. The `tui-commands` requirement "/model changes provider and model in place" already asks for the source. This change makes the code do it, so there is no spec delta (`skip_specs: true`).

## Impact

- **Code (`nar` repo):** `src/nar/cli/commands.py` (`_model`), `src/nar/cli/runner.py` (`switch_model`), `src/nar/agent/loop.py` (`Agent`), and tests.
- **Behavior:** the no-argument output of `/model` is longer. Nothing else changes.
- **Dependencies:** none.
