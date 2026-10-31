# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-69
- **Video timestamps covered:** 15:38:32 (Approval System)
- **Depends on:** RRG-63 `add-agent-loop` (merged in `nar`, its `ToolGate` seam is replaced here), RRG-64 `add-file-tools` (archived; `write_file` and `edit_file` are the tools whose diff is previewed) and RRG-65 `add-shell-tool` (archived; `shell` is the tool whose command is previewed). **Archive order:** this change modifies a requirement introduced by the in-flight change `add-agent-loop` (see Capabilities), so `add-agent-loop` (and, before it, `add-llm-providers`) must be archived before this change is archived.
- **Unblocks:** RRG-74 `add-mcp-client` (MCP tools go through the same approval path without being listed anywhere), RRG-72 `add-hooks`, RRG-73 `add-subagents` (an agent built without a prompter refuses approval-required tools instead of asking) and RRG-78 `add-tui-commands` (`/approval` can switch the level of a running session).
- **Course requirement covered:** human in the loop (approval system).

## Why

Today the agent can change files and run commands, but `nar` cannot ask the user first: the agent loop (RRG-63) refuses every tool that requires approval unless the user opted into `approval = "yolo"` up front, so the safe default is "the agent can read but never act". Human in the loop is a course requirement, and it is what makes `write_file`, `edit_file` and `shell` usable at all: the user must see what the tool is about to do (the command, the diff) and decide, per call, whether it happens.

## What Changes

- **Ask before running.** Before a tool that requires approval runs, `nar` shows what the call will do and asks: **yes**, **no**, or **yes for the whole session**.
  - **What is shown:** for `shell`, the command (and timeout); for `edit_file`, the unified diff the edit would produce; for `write_file`, the target path and either a diff against the existing file or the beginning of the new content; for any other tool (an MCP tool later), the tool name and its arguments. The preview is computed without changing anything.
  - **Session approval** remembers the tool name for the rest of the session ("yes for `edit_file`") and the prompt says so. It lasts for the process (a whole interactive session, or the one request in single mode) and is never saved to disk.
  - **A call that cannot run is not asked about:** when the preview already knows the call would fail (`old_string` not found, a command on the safety list), the model receives that error directly and the user is not interrupted.
- **Autonomy levels** `ask`, `auto-edit` and `yolo`, set with the existing `approval` configuration key and a new `--approval <level>` command-line option:
  - `ask` (default): every approval-required call is asked.
  - `auto-edit`: `write_file` and `edit_file` run without asking when the file is inside the working directory; `shell`, edits outside the working directory and any other approval-required tool are still asked.
  - `yolo`: nothing is asked (unchanged behavior).
- **Rejection goes back to the model.** When the user says no, the tool does not run and the model receives an error result saying the user rejected the call, with the reason when the user typed one (an optional line after "no"), and an instruction not to repeat the same call but to propose something else.
- **Never hang.** When nar cannot ask (standard input or standard error is not a terminal, as in `echo "..." | nar` or `nar "..." 2> log`), approval-required calls are refused with a clear error result saying how to allow them (`--approval`), the same way for every such call. End of input at the prompt counts as "no". Ctrl+C at the prompt cancels the request like anywhere else.
- **Extension point for new tools.** `Tool` gains `preview(arguments, context)` (default: tool name and arguments, so a tool nobody wrote a preview for is still shown correctly) and `is_project_edit(arguments, context)` (default `False`, so unknown tools are asked even at `auto-edit`). Nothing in the approval code lists tool names.
- Replace `approval_gate(level)` in the agent loop with an approval gate that takes the level, a prompter (the terminal one in the CLI, none for headless callers) and the tool context. The loop itself still never prints or reads input.
- Update the README (levels, prompt, `--approval`, non-interactive behavior) and the `shell` section that says the agent only runs the tool at `yolo`.

Out of scope: remembering approvals between sessions or per command/path patterns (allow-lists), approving a subset of a batch of tool calls in one answer, editing the call before approving it, a slash command to change the level during a session (RRG-78), configuration of which tools need approval (`requires_approval` stays a property of each tool), and protection against a file changing between the preview and the execution.

**"Done when" (RRG-69):** in `ask` mode, rejecting an `edit_file` leaves the file intact and the agent proposes something else.

## Capabilities

### New Capabilities

- `approval-system`: when `nar` asks, what the user sees and can answer, what each autonomy level auto-approves, session approval, how a rejection or an impossible-to-ask situation is reported to the model, and how tools take part (preview and project-edit hooks).

### Modified Capabilities

- `agent-loop`: the requirement "Tools that require approval" (introduced by the in-flight change `add-agent-loop` as a temporary rule: run only at `yolo`) is replaced by the approval behavior above. **Archive-order dependency:** `add-agent-loop` must be archived first, because this delta modifies the requirement text as that change wrote it.
- `tool-framework`: new requirements for the tool call preview and for the project-edit declaration that tools expose.
- `edit-file-tool`: the preview of an `edit_file` call is the diff of the edit and reports the same errors as running it.
- `write-file-tool`: the preview of a `write_file` call.
- `shell-tool`: the preview of a `shell` call; the registration scenario "Agent runs it only when approval allows" is rewritten for the new behavior.
- `configuration`: meaning of the three approval levels and the command-line override of `approval`.
- `cli-entrypoint`: the `--approval` option.

## Impact

- **Code (`nar` repo):** new `src/nar/agent/approval.py` (decisions, requests, prompter protocol, `ApprovalGate`), new `src/nar/ui/approval_prompt.py` (terminal prompter); `src/nar/tools/base.py` (`ToolPreview`, `Tool.preview`, `Tool.is_project_edit`), `edit_file.py`, `write_file.py` and `shell.py` (previews); `src/nar/agent/loop.py` and `src/nar/agent/__init__.py` (the gate moves out of the loop module), `src/nar/cli/runner.py` (builds the gate with the terminal prompter), `src/nar/cli/main.py` (`--approval`); README; tests under `tests/`, all offline with a scripted prompter.
- **Dependencies:** none (Rich is already used for the terminal).
- **Configuration:** no new keys; the existing `approval` key (`ask`, `auto-edit`, `yolo`, default `ask`) now has its full meaning and a command-line override.
- **Interfaces other changes rely on:** RRG-74 registers MCP tools with `requires_approval = True` and, optionally, a `preview`; RRG-72 hooks and RRG-73 subagents compose the same `ToolGate` signature; RRG-78 can set `gate.level` to change autonomy mid-session.
- **User-visible:** at the default level, `nar` now pauses on `write_file`, `edit_file` and `shell` with a `[y]es / [n]o / [a]lways this session` question on the terminal; `nar --approval yolo ...` behaves as the old `approval = "yolo"`.
