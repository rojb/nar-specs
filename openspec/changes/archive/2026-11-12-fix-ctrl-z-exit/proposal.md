# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-96
- **Video timestamps covered:** none (found by the real-API end-to-end run on 2026-11-12)
- **Depends on:** none
- **Course requirement covered:** none of the listed ones; it is a correctness fix on the CLI entry point

## Why

The `cli-entrypoint` scenario "End of input" says Ctrl+Z then Enter on Windows ends the session with exit code 0. In a Windows console, the real-API run of 2026-11-12 showed something else: at the `nar>` prompt, Ctrl+Z then Enter sent `^Z` to the model as a request. The console gives the program the character `\x1a` instead of an end-of-file, so the prompt never sees the end of input.

## What Changes

- A line that is only `\x1a`, after whitespace is stripped, counts as end of input. The session ends with exit code 0 and nothing is sent to the model.
- This works for a terminal and for piped input.
- Ctrl+D, Ctrl+C at an empty prompt and `exit` work as before.

Out of scope: other control characters, and changing the prompt itself.

**"Done when" (RRG-96):** in a Windows console, Ctrl+Z then Enter at `nar>` exits with code 0 and no request reaches the model.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

None. The `cli-entrypoint` scenario "End of input" already says Ctrl+Z then Enter ends the session. This change makes the code do it, so there is no spec delta (`skip_specs: true`).

## Impact

- **Code (`nar` repo):** `src/nar/cli/repl.py` (`run`), tests. `src/nar/cli/prompt.py` (`read_line`) only reads the line and is not changed.
- **Behavior:** a request that is only `^Z` no longer reaches the model.
- **Dependencies:** none.
