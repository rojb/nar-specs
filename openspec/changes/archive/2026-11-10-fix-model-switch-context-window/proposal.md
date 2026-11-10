# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-90 (sub-issue of RRG-78)
- **Video timestamps covered:** none (found on 2026-11-10 while testing RRG-89)
- **Depends on:** RRG-70 `add-context-compaction`, RRG-81 `fix-compaction-model-switch` and RRG-89 `add-provider-fallback` (all merged)
- **Course requirement covered:** context management and selectable model provider, both only as a fix

## Why

`ContextLimits` is built once at startup, from the first provider. `/model`, and the switch that RRG-89 offers, change the provider of the agent and the compactor (RRG-81) but not the limit. Starting on Ollama (4,096 tokens by default) and switching to DeepSeek, `nar` still showed `context: 2,702 / 4,096 tokens (65%)`. With 128,000 tokens available, it would compact far too early. The other way round, from 128,000 to 4,096, it would go over the new model's context.

## What Changes

- A switch of provider or model also takes the new provider's default context window, unless `context_window` was set in a configuration file or on the command line. The threshold does not change.
- If the conversation is already over the new limit, it is reduced before the next model call, with the usual pruning and compaction.

**"Done when" (RRG-90):** after `/model deepseek` starting on Ollama, `context:` shows `/ 128,000 tokens`.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `context-compaction`: a new requirement that the context limit follows the provider in use. ADDED.

## Impact

- **Code (`nar` repo):** `src/nar/config/models.py` (whether `context_window` was set), `src/nar/context/compaction.py` (`ContextLimits.fixed`), `src/nar/cli/runner.py` (`create_agent`, `switch_model`), and tests.
- **Dependencies:** none new.
