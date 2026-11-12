# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-94
- **Video timestamps covered:** none (found by the real-API end-to-end run on 2026-11-12)
- **Depends on:** none (corrects `fix-hook-block-exit-code`, merged and archived)
- **Course requirement covered:** none of the listed ones; it is a correctness fix on the hook mechanism

## Why

A `before_tool` hook that exits with the block code `2` blocks the call, as the `hooks` spec says. But `nar` also prints `warning: hook before_tool failed with exit code 2: <command>`. A refusal is not a failure. The warning makes a working policy look broken, and it sits next to the "refused by a hook" notice that already explains what happened.

## What Changes

- A `before_tool` outcome that blocks the call produces no failure warning.
- The "refused by a hook" notice and the hook's own text are still shown.
- Every other case warns exactly as today: exit code 1 or any other non-zero code (with `did not block the call`), a hook that cannot start, and a timeout.

Out of scope: changing the block code, the wording of the other warnings, and the other four events.

**"Done when" (RRG-94):**
- a `before_tool` hook that exits with 2 blocks the call and no `warning: hook ... failed` line appears;
- a hook that exits with 1 still warns that it did not block.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `hooks`: "A before_tool hook can refuse the call" says a refusal produces no warning, with a new scenario.

## Impact

- **Code (`nar` repo):** `src/nar/hooks/runner.py` (`HookRunner._report`), `tests/test_hooks_runner.py`.
- **Behavior:** one warning line less when a hook refuses a call. Nothing else changes.
- **Dependencies:** none.
- **Specs:** one MODIFIED requirement in `hooks`.
