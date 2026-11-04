# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-68
- **Video timestamps covered:** 11:17:06 (Todos/Planning Tool), 11:39:20 (Memory Tool), 12:06:43 (Updating the System Prompt)
- **Depends on:** RRG-63 `add-agent-loop` (merged; not archived yet, see Impact)
- **Unblocks:** RRG-76 `add-sessions` can later persist the todo list with the session
- **Course requirement covered:** none directly; the memory supports *context management* by keeping lasting facts out of the conversation history

## Why

On multi-step tasks the agent loses track of what it has done and what is left, and the user cannot see its plan. Between sessions it forgets everything: a preference the user stated yesterday ("answer in Spanish", "we use pytest, not unittest") has to be repeated every time. The agent needs a visible task list and a small persistent memory, and the system prompt has to tell it to use both.

## What Changes

- Add the built-in **`todos`** tool: the agent writes its whole task list in one call, each task with a status (`pending`, `in_progress`, `completed`), with at most one task in progress. The list lives for the session and is shown to the user as a checklist every time it changes.
- Add the built-in **`memory`** tool: the agent saves, deletes and lists short named facts (a key and a value) about the user or the project. They are stored in `~/.nar/memory.json` (`NAR_HOME` respected) and survive between sessions, with limits on key and value length and on the number of entries.
- **Inject saved memory** into the system prompt at the start of every session, so the agent knows it without calling a tool.
- **Update the system prompt**: plan multi-step tasks with `todos` and keep it current; save lasting preferences and facts with `memory`.
- Extend the **tool activity display** so a tool result can carry extra text shown to the user below the success line (used for the checklist), the same way a diff is shown today.
- Neither tool requires approval.

Out of scope: saving the todo list across sessions (RRG-76 sessions), per-project memory files, editing memory from a slash command (RRG-78), automatic memory extraction without the model calling the tool.

**"Done when" (RRG-68):** in a new session the agent remembers a preference saved in an earlier session.

## Capabilities

### New Capabilities

- `todos-tool`: behavior of the built-in `todos` tool: the list, statuses, the single in-progress rule and how it is shown.
- `memory-tool`: behavior of the built-in `memory` tool: saving, deleting and listing entries, the storage file and its limits, errors, and the injection of saved memory at session start.

### Modified Capabilities

- `tool-framework`: the tool activity display requirement gains showing extra text that a result carries for the user.

## Impact

- **Code (`nar` repo):** new `src/nar/tools/todos.py`, `src/nar/tools/memory.py` (tool and `MemoryStore`); `ToolResult` gains an optional `display` field (`src/nar/tools/base.py`) and `ToolRenderer` prints it (`src/nar/ui/tool_view.py`); registration in `src/nar/tools/builtin.py`; `src/nar/agent/prompt.py` gains planning/memory guidelines and a memory section; `src/nar/cli/runner.py` loads the memory when building the agent; README.
- **User filesystem:** writes `~/.nar/memory.json` (or `$NAR_HOME/memory.json`) when the agent saves something.
- **Dependencies:** none new.
- **Specs:** this change does not touch `agent-loop` or `agent-context`, so it can be archived before or after `add-agent-loop`.
