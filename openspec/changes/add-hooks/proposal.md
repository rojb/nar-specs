# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-72
- **Video timestamps covered:** 16:32:00 (Hooks)
- **Depends on:** RRG-63 `add-agent-loop` (merged in `nar`; not archived yet, see Impact)
- **Unblocks:** nothing; it reuses the loop's events and needs no other change
- **Course requirement covered:** none of the listed course requirements; it adds the hook mechanism of the course on top of the agent loop

## Why

Everything `nar` does to a repository happens inside a tool call, and the only way to react to it is
to ask the model to do it. That has three costs: it spends a model call on a task that is mechanical
(run the formatter after an edit), it depends on the model remembering to do it, and a project cannot
enforce a rule at all — a hook that refuses an edit outside `src/` is a policy, but asking the model
to respect one is a suggestion.

The pieces are already there: `Agent.gate` intercepts a call before its tool runs (RRG-69), the loop
yields events after each one, and `nar.tools.shell_process.run_command` already runs a command with a
timeout and kills its whole process tree. What is missing is a way for the project to say "when this
event happens, run this command".

## What Changes

- **Five events:** `before_agent`, `before_tool`, `after_tool`, `after_agent` and `on_error`.
- **Configured as shell commands:** `[hooks]` in `.nar/config.toml` or `~/.nar/config.toml`, one list
  of command strings per event. They run in the project folder, through the same shell `nar` already
  uses for its `shell` tool, and receive the event as JSON on standard input.
- **`before_tool` can block:** a non-zero exit code stops the call and its output becomes the result
  the model receives, so the model learns the rule instead of being left guessing.
- **Nothing a hook does can break `nar`:** a hook that is missing, crashes or times out warns and the
  request goes on, and a `before_tool` hook that cannot run does not block anything.
- **Visible output:** what a hook prints is shown on standard error, as dim lines, so the user sees
  the formatter or the linter as it runs.
- **Documented example:** a `ruff` hook after `edit_file` on a `.py` file, in the README.

Out of scope: hooks written as Python functions, a hook that edits the arguments of a call, hooks
that can change the model's answer, hooks per tool name instead of per event, running hooks in
parallel, and a hook timeout different per event.

**"Done when" (RRG-72):** *"el hook de ejemplo corre automáticamente tras una edición y su salida se
ve en la TUI"* — quoted from the Linear issue, which is written in Spanish. In English: the example
hook runs by itself after an edit and its output is visible in the TUI.

## Capabilities

### New Capabilities

- `hooks`: the five events, the JSON they carry, how a hook is configured and run, blocking on
  `before_tool`, isolation of a failing hook, and what the user sees.

### Modified Capabilities

None. The timeout is a new `configuration` requirement (ADDED next to the other settings), so the
change adds requirements only and can be archived in any order.

## Impact

- **Code (`nar` repo):** new `src/nar/hooks/` (`HookRunner`, the event payloads, the shell call);
  `src/nar/agent/loop.py` (`Agent(hooks=...)`, the five call sites); `src/nar/agent/events.py`
  (`HookRan`, `HookBlocked`, `HookFailed`); `src/nar/ui/agent_view.py` (their lines);
  `src/nar/cli/runner.py` (builds the runner from the config); `src/nar/config/models.py`
  (`hooks`, `hook_timeout`); README.
- **Existing behavior:** with no `[hooks]` table nothing runs, nothing is imported and the loop is
  exactly what RRG-63 left.
- **Dependencies:** none new; `nar.tools.shell_process` is reused instead of a second process helper.
- **Cost:** one process per configured hook per matching event, started synchronously, so a slow
  hook slows the request. That is why the timeout exists and why the default is 10 seconds.
- **Specs:** one new capability plus one ADDED requirement in `configuration`; no MODIFIED
  requirement, so no archive ordering constraint against `add-agent-loop` or `add-mcp-client`.
