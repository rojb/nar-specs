# Design

## Context

`create_agent` builds `Compactor(provider, limits)`, and `switch_model` assigns `agent.provider` only when the new provider could be created. `TaskTool` builds a new `Compactor` from `parent.provider` on each call, so subagents are not affected.

## Decisions

### 1. Update the compactor in `switch_model`
`src/nar/cli/runner.py` (MODIFIED): after `agent.provider = provider`, set `agent.compactor.provider = provider` when there is a compactor. No new configuration keys. **Alternative rejected:** having `Compactor` read the provider from the agent, which would make it depend on `Agent` for one assignment and would change the signature RRG-70 and RRG-73 use.

## Risks / Trade-offs

- **A later place that changes the provider could forget this again** → `switch_model` is the only one today, and the test covers the behavior through `/model`.

## Migration Plan

None.

## Open Questions

None.
