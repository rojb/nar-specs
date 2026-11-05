# Design

## Context

`SlashContext` carries two Rich consoles: `console` for standard output and `err` for standard error.
The base commands (`/help`, `/model`, `/approval`, `/config`, `/tools`, `/tokens`, `/compact`,
`/resume`) write through `err`, and so do the notes, warnings and history. Three commands — `/mcp`
(`_mcp_command`), `/sessions` (`_sessions_command`) and `/checkpoints` (`_checkpoints_command`) — pass
`context.console` to their views instead, because the changes that added them (RRG-74, RRG-76, RRG-77)
decided that the answer belonged on standard output.

The `tui-commands` spec (RRG-78) later stated the global rule that no command writes to standard
output. Both behaviours are implemented and tested, so the store contains two contradictory promises;
the smoke test of 2026-11-05 found the consequence.

## Goals / Non-Goals

**Goals:**
- One rule for every command: the answer goes to standard error, standard output keeps the answer the
  model produced.
- A test that fails if any of the three commands goes back to standard output.

**Non-Goals:**
- Changing the view functions: `render_mcp_status`, `render_sessions` and `render_checkpoints` keep
  taking the console to print to, so their own tests stay valid and a caller that wants another stream
  can pass one.
- Touching the banner and the prompt, which belong on standard output.
- Rewriting the archived designs of RRG-74/76/77: they record what those changes decided, and the
  archive is history, not policy.

## Decisions

### 1. Layout

```
src/nar/cli/slash.py     MODIFIED  the three handlers pass context.err
tests/test_slash.py      MODIFIED  six tests assert stderr and an empty stdout
tests/test_cli.py        MODIFIED  three tests assert stderr and an empty stdout
README.md                unchanged already says commands answer on standard error
openspec/specs/tui-commands/spec.md  MODIFIED after archive: scenarios naming the three commands
```

### 2. The handler chooses the stream, not the view

The views already receive the console as a parameter, and each one takes an `err` console for
problems. Passing `context.err` for both keeps a single source of truth (the router) and avoids a new
default inside the UI layer that a later caller could forget. The redundant `err=context.err` in
`render_mcp_status` and `render_sessions` is deliberate: the answer and the problems go to the same
stream, so the call reads the same as the other commands.

### 3. The rule is locked with scenarios, not with prose

The requirement already said "no command SHALL write its output to standard output", and that was not
enough: the three commands were written before it and nobody re-read the older ones. The delta adds a
scenario that names `/mcp`, `/sessions` and `/checkpoints` and a scenario for the redirected run, so a
reader of the capability sees the cases that actually broke.

## Risks / Trade-offs

- **A script that parsed `/mcp` or `/sessions` from standard output breaks** → the README already
  documents standard error, no consumer is known, and the spec's promise is the one that has to hold.
  The change is one `printf | nar > file` away from being noticed.
- **The tests could assert the stream twice** (unit and CLI) → deliberate: the unit tests pin the
  router, the CLI tests pin the process boundary where the original bug was observable.
- **The archive and the code now disagree** → expected; the archived designs are a record of those
  changes, and the current spec plus this change are the authority.
