# Design

## Context

`NarConfig._default_model_for_provider` fills `context_window` from `DEFAULT_CONTEXT_WINDOWS[provider]` when the files and flags leave it unset. After that, the config cannot tell a default from a value the user wrote. `create_agent` builds `ContextLimits(config.context_window, config.context_threshold)` once. `switch_model` replaces the provider of the agent and the compactor (RRG-81), but not `compactor.limits`. `TaskTool` builds each subagent's compactor from `parent.compactor.limits`.

## Decisions

### 1. Remember whether the window was set
`src/nar/config/models.py` (MODIFIED): a private attribute `_context_window_set`. The after-validator records it before it fills the default, and a read-only property `context_window_set` exposes it. It is not a field, so `--show-config` and the loader do not change.

### 2. `ContextLimits.fixed`
`src/nar/context/compaction.py` (MODIFIED): a new field `fixed: bool = False` on the frozen dataclass. `create_agent` (`src/nar/cli/runner.py`, MODIFIED) passes `fixed=config.context_window_set`.

### 3. `switch_model` updates the limit
`src/nar/cli/runner.py` (MODIFIED): after a successful switch, when `compactor.limits.fixed` is false, it sets `compactor.limits = dataclasses.replace(limits, window=DEFAULT_CONTEXT_WINDOWS[provider_name])`.
- The RRG-89 fallback goes through `switch_model`, so it is covered too.
- The loop reads `compactor.limits` before every model call, so a conversation over the new limit is pruned or compacted before it reaches the new provider.
- Subagents read `parent.compactor.limits` when they start.

No new configuration keys.

## Risks / Trade-offs

- **The default is per provider, not per model** → this matches how the startup default already works. A user who needs another window sets `context_window`, and it is then kept on every switch.
- **Switching to a smaller window may compact at once** → that is the point: otherwise the next request would go over the new model's context.

## Migration Plan

None.

## Open Questions

None.
