# Design

## Context

`Agent.run` (`src/nar/agent/loop.py`) is a generator: it yields an event before each model call, after
each tool call, and at the end. `Agent.gate` (RRG-69) is consulted before a tool runs and can return a
`ToolResult` instead. `ToolResult` carries `error_kind` (RRG-71), and `FailureKind.REFUSED` is the one
kind the loop does not wrap in the failure envelope, which is exactly the shape a blocked call needs.

`nar.tools.shell_process` already knows how to run a command with a timeout and kill the whole process
tree on Windows and on POSIX (`find_shell`, `build_argv`, `run_command`). Hooks must reuse it rather
than grow a second process helper: two ways to spawn a shell in a Windows codebase is one too many.

## Goals / Non-Goals

**Goals:**
- A project can run its own command around any of the five events without touching `nar`'s code.
- A broken hook is a warning, never a broken agent.
- The blocking case delivers the hook's own text to the model, because that text is the rule.

**Non-Goals:**
- Hooks that rewrite arguments or results, Python hooks, per-tool configuration, parallelism,
  sandboxing of what a hook may do, or a timeout per event.

## Decisions

### 1. Layout
```
src/nar/hooks/__init__.py    NEW       re-exports HookRunner, HookEvent, HookOutcome
src/nar/hooks/runner.py      NEW       HookRunner, payload builders, run_hook()
src/nar/config/models.py     MODIFIED  hooks: dict[str, list[str]] validated, hook_timeout: float = 10
src/nar/agent/events.py      MODIFIED  HookRan, HookBlocked, HookFailed
src/nar/agent/loop.py        MODIFIED  Agent(hooks=...), the five call sites
src/nar/ui/agent_view.py     MODIFIED  shows the three hook events
src/nar/cli/runner.py        MODIFIED  builds the HookRunner from the config
README.md                    MODIFIED  the ruff example
```

### 2. Configuration
| Key | Type | Default |
|---|---|---|
| `hooks` | table, keys `before_agent`, `before_tool`, `after_tool`, `after_agent`, `on_error`, each a list of command strings | empty |
| `hook_timeout` | number of seconds, 1 to 600 | `10` |

`HookEvent` (`StrEnum`) holds the five names, and the model validator rejects any other key with the
usual "unknown setting" wording. **Why commands and not tables per hook:** a hook is one command with
one contract (JSON in, exit code out), and a table invites per-hook options that this change does not
need. **Why a single timeout:** a slow hook is a slow hook; per-event timeouts are a knob without a
use case today.

### 3. Payloads
One JSON object per hook, on standard input, one line:

| Event | Fields |
|---|---|
| `before_agent` | `event`, `prompt`, `cwd` |
| `before_tool` | `event`, `tool`, `arguments`, `cwd` |
| `after_tool` | `event`, `tool`, `arguments`, `is_error`, `output`, `cwd` |
| `after_agent` | `event`, `prompt`, `outcome`, `turns`, `cwd` |
| `on_error` | `event`, `where`, `message`, `cwd` |

`output` is cut to 4,000 characters with a trailing note, because a hook inspecting a 30,000-character
tool result is a hook that wanted a file. `outcome` is `finished`, `turn_limit`, `loop`, `cancelled` or
`error`. **Why JSON on stdin and not arguments or environment:** arguments hit Windows quoting, the
environment has size limits, and stdin survives both; it is also what the course does.

### 4. Running a hook
`run_hook(command, payload, cwd, timeout, shell)` builds the argv with `build_argv`, starts the process
with `subprocess.PIPE` on all three streams, writes the JSON, and reads the rest with the same capture
helper `nar` uses for its `shell` tool, so a hook that spawns children is killed as a tree on timeout
and on Ctrl+C. The hook inherits the environment and runs in the project folder, because a linter
needs to resolve the project's configuration.

`HookRunner` holds the configuration, the working directory, and two callbacks (`note`, `warn`) so it
never imports `rich`. It exposes `before_agent`, `before_tool`, `after_tool`, `after_agent`,
`on_error`, and each returns what the loop needs: a `ToolResult` to use instead of running the tool
when a `before_tool` hook blocked the call, otherwise `None`. **Why callbacks and not the view:** the
same reason as the loop: `nar.hooks` must stay testable without a terminal.

### 5. Blocking and isolation
- `before_tool` exit code 0 → run the tool. Non-zero → `ToolResult.error(text, FailureKind.REFUSED)`
  with the hook's stdout and stderr joined, prefixed by `the before_tool hook refused this call: `, so
  the model reads the rule and the envelope stays out of the way.
- A hook that cannot start (missing command), or that times out, warns and does not block. **Why
  fail-open:** a project that mistypes a path must not lose the ability to edit files; the warning is
  loud, and the alternative bricks the agent on a typo.
- A non-zero exit on any other event warns, because those events cannot stop anything anyway.
- Output shown to the user is cut to 2,000 characters per hook. The model never sees it, except the
  refusal text above.

### 6. Placement in the loop
```
before_agent(prompt)                     -> HookRan
for each turn:
    _manage_context, model call
    for each tool call:
        gate (approval)                  -> may refuse, then no hooks run
        hooks.before_tool(call)          -> HookBlocked or HookRan; may return the result
        tool runs                        (unless blocked)
        hooks.after_tool(call, result)    -> HookRan
        hooks.on_error(...)              only when the result is an error -> HookRan
    loop detection
after_agent(prompt, outcome, turns)       -> HookRan
```
`after_agent` runs from the `finally` of `Agent.run`, so a cancellation, a provider failure and a loop
abort all report the same way; the outcome variable is set before each exit path. `on_error` also fires
for a provider failure, right before it propagates.

### 7. Display
`AgentView` prints, on standard error, literally: `hook after_tool: ruff check .` followed by the
hook's output as dim lines when it printed something; `hook before_tool refused the call: <first line>`
as a warning; `warning: hook before_tool could not run: <reason>`. Nothing is written to standard
output, so `nar "..." > answer.txt` still keeps only the answer.

## Risks / Trade-offs

- **A slow hook slows every request** → the 10-second default and the timeout kill protect the loop;
  the README says hooks run synchronously.
- **Hooks are arbitrary shell commands from a project file** → the same trust boundary as
  `[mcp_servers]` and the shell tool: the project file is already code the user chose to run. The
  README says so, and `nar` never runs hooks from a file it did not load.
- **Fail-open can hide a broken rule** → the warning names the command and the event, and the
  alternative (fail-closed) turns a typo into a `nar` that cannot edit files.
- **The model receives hook text only when blocked** → intended: a formatter's output is for the user,
  and feeding it to the model would spend a turn on it.

## Migration Plan

Additive. No `[hooks]` table means nothing new runs, and no stored data changes.

## Open Questions

None.
