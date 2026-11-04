# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-73
- **Video timestamps covered:** 12:15:41 (Subagents)
- **Depends on:** RRG-63 `add-agent-loop` (merged; not archived yet, see Impact)
- **Unblocks:** nothing
- **Course requirement covered:** subagent research (`docs/SUBAGENTES.md`)

## Why

Exploring a repository fills the conversation with long tool results that are only useful for reaching one conclusion, and every later model call carries them. Compaction (RRG-70) reduces a history that has already grown; a subagent stops it from growing. The research and the comparison with Claude Code, Codex and OpenCode are in `docs/SUBAGENTES.md`.

## What Changes

- **`task` tool:** the main agent launches a subagent with its own conversation, a prompt and a subset of tools, and receives only its final answer.
- **Two predefined types:** `explorer` (read only: `read_file`, `grep`, `glob`, `list_dir`) and `general` (the other tools, with approvals).
- **Limits:** a configurable turn limit (`subagent_max_turns`, default 15), and no nesting, because a subagent does not have `task`.
- **Same controls:** a subagent's calls go through the same approval gate and the same hooks as the main agent's. "Always allow" answers are shared.
- **Folded activity:** while a subagent works, one line shows its progress; when it ends, one line remains, and its answer text never reaches standard output.
- **System prompt:** the main agent is told when to delegate.

Out of scope: subagents running in parallel or in the background, subagents defined by the user, a different model per subagent, saving a subagent's messages or letting the user open them, resuming a subagent.

**"Done when" (RRG-73):** "explore this repo and explain the architecture" launches an `explorer`, and the main agent's context only grows with the summary.

## Capabilities

### New Capabilities

- `subagents`: the `task` tool, the two types, their isolation and limits, approvals and hooks, results and failures, and how the activity is shown.

### Modified Capabilities

- `configuration`: a new requirement for `subagent_max_turns`, ADDED next to "Initial settings".

## Impact

- **Code (`nar` repo):** new `src/nar/agent/subagent.py`; `src/nar/agent/prompt.py`, `src/nar/tools/registry.py` (`subset`), `src/nar/config/models.py`, `src/nar/cli/runner.py`, `src/nar/ui/agent_view.py`; README.
- **Documentation (`nar-specs`):** new `docs/SUBAGENTES.md`.
- **Cost:** a subagent makes its own model calls, so a delegated task can use more tokens in total even though the main context stays small. The README says so.
- **Dependencies:** none new.
- **Specs:** a new capability and an ADDED requirement, so it can be archived before or after `add-agent-loop`.
