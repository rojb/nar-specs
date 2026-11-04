# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-77
- **Video timestamps covered:** 18:16:12 (Sessions vs Checkpoints), 18:55:01 (Checkpoint Management)
- **Depends on:** RRG-76 `add-sessions` (merged; not archived yet, see Impact)
- **Unblocks:** nothing
- **Course requirement covered:** none directly; it extends sessions with history

## Why

A session keeps one line of conversation. When a request goes the wrong way, the user can only keep going with the bad turns in the context, or start again. A checkpoint gives a known point inside the session to go back to.

The difference matters and has to be documented: the **session** is the whole conversation; a **checkpoint** is a point inside it that can be returned to.

## What Changes

- **`/checkpoint [name]`** saves the state of the conversation inside the current session, with a number and an optional name.
- **`/checkpoints`** lists the checkpoints of the current session.
- **`/restore <id or name>`** returns the conversation to that point: later messages are discarded and the agent continues from there. Files changed by tools are not restored, and `nar` says so every time.
- **Checkpoints last with the session:** they survive closing `nar` and are available again after `nar --resume` or `/resume`.
- **A later checkpoint stays after a restore,** so a restore can be undone by restoring the later one.

Out of scope: restoring files or undoing shell commands, deleting or renaming checkpoints, automatic checkpoints, restoring the `todos` list or "always allow" answers, checkpoints in single mode.

**"Done when" (RRG-77):** restoring a checkpoint discards the later messages and the agent continues from there.

## Capabilities

### New Capabilities

- `checkpoints`: saving, listing and restoring checkpoints, how long they last, failures, and the difference from a session.

### Modified Capabilities

None.

## Impact

- **Code (`nar` repo):** new `src/nar/session/checkpoints.py`; `src/nar/session/store.py` (shared helpers become public), `src/nar/session/recorder.py`, `src/nar/ui/session_view.py`, `src/nar/cli/slash.py`; README. The commands use `register_command`, the seam RRG-78 keeps.
- **User filesystem:** writes `~/.nar/sessions/checkpoints/<session id>.json`, readable only by the user on Linux and macOS. Like sessions, these files hold the conversation in plain text and `nar` never removes them.
- **Dependencies:** none new.
- **Specs:** only a new capability, so it can be archived before or after `add-sessions`.
