# Design

## Context

The REPL loop in `src/nar/cli/repl.py` (`run`) reads a line with `read_line` from `src/nar/cli/prompt.py`, strips it, and sends it to the agent unless it is empty, an exit word, or a slash command. End of input is handled by catching `EOFError` and `KeyboardInterrupt` around `read_line`; both print a newline and return, so the command exits with code 0.

In a Windows console, Ctrl+Z then Enter does not raise `EOFError` in this path. The line arrives as the single character `\x1a` (it shows as `^Z`), and `run` treats it as a normal request. Python's `str.strip()` does not remove `\x1a`, because it is not whitespace.

`read_line` has two paths: `input(prompt)` when standard input is not a terminal, and a `prompt_toolkit` session when it is. Both return the line as text, so one check after the read covers both.

## Goals / Non-Goals

**Goals:**
- A line that is only `\x1a` ends the session with exit code 0 and sends nothing.
- It works for a terminal and for piped input.

**Non-Goals:**
- Handling other control characters.
- Changing `read_line` or the prompt.

## Decisions

### 1. Layout
```
src/nar/cli/repl.py      MODIFIED  run() treats a line that is only "\x1a" as end of input
tests/test_cli.py        MODIFIED  piped "\x1a" ends with exit code 0 and no request; "\x1a" with spaces too
```
`src/nar/cli/prompt.py` is read for context and stays as it is.

### 2. Same exit as EOF
After `line = line.strip()`, `run` checks `line == "\x1a"`. On a match it does what the `EOFError` branch does: `click.echo()` and `return`. The session ends the same way, the recorder is closed as for any other exit, and no request or hook runs. The check comes before the empty-line test is irrelevant: a `\x1a` line is not empty, and it must come before slash commands and the agent call.

A `\x1a` that is part of a longer line is ordinary text and is sent as before.

### 3. No new config keys
There are no new keys in `~/.nar/config.toml` or `.nar/config.toml`.

## Risks / Trade-offs

- **A user who really wants to send `^Z` as a request** → not a real use case, and the spec already says this key combination ends the session.
- **The unit tests cannot press Ctrl+Z in a Windows console** → they pipe the character; the manual verification covers the real console.

## Migration Plan

None.

## Open Questions

None.
