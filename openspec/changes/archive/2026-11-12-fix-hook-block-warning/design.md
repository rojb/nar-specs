# Design

## Context

`HookRunner._report` (`src/nar/hooks/runner.py`) runs after every hook and prints a warning for any outcome that is not a clean success. Its non-zero branch prints `warning: hook <event> failed with exit code <N>...` and then the hook's output. That branch does not look at `outcome.blocks`, so a `before_tool` hook that exits with the block code `2` gets a failure warning even though `before_tool` is about to refuse the call on purpose.

`HookOutcome.blocks` already says whether an outcome is a refusal: no startup error, no timeout, and `exit_code == BLOCK_EXIT_CODE`. `HookRunner.before_tool` uses it to build the refused `ToolResult`. `_report` is the only place that needs to use it too.

## Goals / Non-Goals

**Goals:**
- A refusal prints no failure warning.
- The hook's text still reaches the user, and the refused result still reaches the model.
- Every other outcome keeps today's warning text.

**Non-Goals:**
- Changing `BLOCK_EXIT_CODE` or the wording of the other warnings.
- Changing the other four events. `HookOutcome.blocks` is only about `before_tool`; other events do not refuse calls.

## Decisions

### 1. Layout
```
src/nar/hooks/runner.py     MODIFIED  HookRunner._report returns early for an outcome that blocks
tests/test_hooks_runner.py  MODIFIED  a refusal writes no warning; exit 1, a missing command and a timeout still warn
```

### 2. Return early on a block
In `_report`, an outcome with `outcome.event is HookEvent.BEFORE_TOOL and outcome.blocks` shows the hook's output with `_show` and returns, before the `exit_code != 0` branch. The "refused by a hook" text is not printed here: it is the error result that `before_tool` returns, which the agent view already renders. Showing the output keeps the hook's own sentence visible, as today.

The check includes the event because `blocks` only looks at the exit code, and an `after_tool` hook that exits with 2 is still a failure that must warn.

### 3. No new config keys
There are no new keys in `~/.nar/config.toml` or `.nar/config.toml`.

## Risks / Trade-offs

- **The user sees the hook text and the refusal notice, but no line saying "hook ran"** → same as a hook that exits 0 with no output; the refusal notice is enough.
- **Tests that count warnings for exit 2** → they are updated in the same commit.

## Migration Plan

None.

## Open Questions

None.
