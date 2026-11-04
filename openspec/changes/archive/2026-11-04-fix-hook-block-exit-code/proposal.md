# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-82
- **Video timestamps covered:** 16:32:00 (Hooks) — the same material as RRG-72; this change corrects the contract it introduced
- **Depends on:** RRG-72 `add-hooks` (merged and archived)
- **Unblocks:** nothing; it restores a promise the archived `hooks` spec already makes
- **Course requirement covered:** none of the listed ones; it is a correctness fix on the hook mechanism

## Why

The archived `hooks` spec promises that a hook which cannot run produces a warning and never blocks a
call, because a mistyped path must not leave a project unable to edit files. The manual verification of
RRG-72 (2026-11-04) showed that the promise only holds for the two cases the tests covered: an
executable that does not exist (`OSError`) and a timeout.

With a real shell it breaks. A `before_tool` hook configured as `this-command-does-not-exist` starts
PowerShell, PowerShell does not find the command and exits with **1**, and the runner reads that 1 as a
refusal: every call in the session is blocked by a typo. The block code and the "the hook is broken"
code are the same number, so the runner cannot tell them apart.

## What Changes

- **A dedicated block code.** `before_tool` blocks the call only when the hook exits with code `2`,
  documented in the README and in the spec.
- **Everything else fails open.** Any other non-zero code, a hook that cannot be started and a timeout
  produce a warning and the call runs.
- **The warning says which it was**, so a user whose policy silently stopped applying sees it: the
  message names the event, the command, the exit code and says that the call was not blocked.

Out of scope: a configuration key for the block code, per-shell detection of "command not found",
hooks that block by printing something instead of by exit code, and changing what the other four
events do with a non-zero code (they already only warn).

**"Done when" (RRG-82):** a `before_tool` hook pointing at a command that does not exist warns and
lets the call run, while a hook that exits with `2` still blocks it.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `hooks`: "A before_tool hook can refuse the call" now names the block code, and "A failing hook
  never breaks the request" states that any other non-zero code never blocks.

## Impact

- **Code (`nar` repo):** `src/nar/hooks/runner.py` (`BLOCK_EXIT_CODE`, the `before_tool` decision and
  the two warnings), `tests/test_hooks_runner.py` (which currently blocks on 1), README.
- **Behavior:** a project whose `before_tool` hook relied on exit code 1 to block must change it to 2.
  Nothing else in the loop changes: the blocked result is still `FailureKind.REFUSED`, hooks still run
  after approval, and the other four events never blocked.
- **Dependencies:** none.
- **Specs:** two MODIFIED requirements in `hooks`; no ordering constraint, since that capability
  already exists in the store.
