# Design

## Context

`HookRunner.before_tool` (`src/nar/hooks/runner.py`) turns every `HookOutcome` with `failed` — meaning
no startup error, no timeout, and a non-zero exit code — into a `ToolResult` with
`FailureKind.REFUSED`. That conflates "the hook said no" with "the hook is broken", and the manual run
of 2026-11-04 proved the two are indistinguishable today: PowerShell reports
`CommandNotFoundException` with exit code **1**, which is also the conventional "refuse" code.

`HookOutcome` already carries the three cases apart (`error`, `timed_out`, `exit_code`), so the fix is a
decision in one place, not new plumbing.

## Goals / Non-Goals

**Goals:**
- A hook that cannot run never takes the agent's tools away.
- A deliberate refusal stays a single, obvious thing for the project author.

**Non-Goals:**
- Detecting "command not found" per shell (`127`, `CommandNotFoundException`, exit 1 again on some
  hosts): fragile, and unnecessary once the block code is explicit.
- A setting for the block code; a constant with a name and a documented reason is enough for one
  number.
- Changing the other four events, which never blocked.

## Decisions

### 1. Layout
```
src/nar/hooks/runner.py     MODIFIED  BLOCK_EXIT_CODE = 2; before_tool blocks only on it
tests/test_hooks_runner.py  MODIFIED  blocking tests use 2; the exit-1 and missing-command cases assert
                                      that the call runs; one real-shell regression test
README.md                   MODIFIED  the block code, and why 1 is not it
```

### 2. The block code is `2`
`before_tool` blocks only when the outcome has no startup error, no timeout and `exit_code == 2`. Every
other outcome lets the call run and warns. **Why 2:** the block code has to be a number a broken hook
does not produce by accident. `1` is produced by every failing command, including the shell's own
"command not found" (PowerShell) and a syntax error; `127` is bash-specific; `2` is what the shell
returns for a misuse, which a typo cannot reach because the shell catches the typo first. It is also the
code the rest of the Unix world uses for "stop, this is a usage error", so it reads as deliberate.

### 3. The warnings say what happened
Two different messages, because the user has to know whether the policy applied:
- blocked: `hook before_tool refused the call: <first line of the hook's output>`
- broken: `warning: hook before_tool failed with exit code <N> and did not block: <command>`, and for a
  timeout or a startup error the existing wording plus `did not block`.

Today both end up as `failed with exit code N`, which is why the manual run looked like a policy that
worked instead of a typo that locked everything.

## Risks / Trade-offs

- **History-breaking for a one-day-old feature** → the only projects with hooks are this repo's README
  example (`after_tool`, unaffected) and any experiment of ours; the README and the archived spec are
  the two places that mention blocking, and both change in this commit.
- **A hook that means "refuse" but exits 1 now silently does nothing** → it warns loudly, naming the
  exit code and saying it did not block, which is the whole point of the fix.
- **A hook that legitimately fails with 2 while meaning something else** → it blocks; unlikely, and the
  model still receives the hook's own text.

## Migration Plan

Anyone with a `before_tool` hook that blocks must change its exit code from 1 to 2. The README states
it; nothing stored changes.

## Open Questions

None.
