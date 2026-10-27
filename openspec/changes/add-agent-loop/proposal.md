# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-63
- **Video timestamps covered:** 01:15:05 (Run Single Mode, Agentic Loop & Rich), 02:05:35 (Context Manager + System Prompt), 04:34:55 (Agentic Loop: running tools, handling context), 06:13:08 (Run Interactive Mode), 06:29:22 (Turns in the Agentic Loop)
- **Depends on:** RRG-61 `add-llm-providers` (merged), RRG-62 `add-tool-framework` (archived)
- **Unblocks:** Phases 3, 4 and 5: RRG-68, RRG-69, RRG-70, RRG-71, RRG-72, RRG-73, RRG-76, RRG-78 (and the agent-driven checks of RRG-64/65)
- **Course requirement covered:** *context management*, part 1 (conversation history, token counting, system prompt + `AGENTS.md`); compaction and pruning are RRG-70.

## Why

`nar` can talk to a model (RRG-61) and run tools (RRG-62), but nothing connects the two: single mode only prints one answer, and interactive mode is still a placeholder that echoes lines. A coding agent needs the loop that lets the model ask for tools, see their results and keep going until it has an answer, with the conversation kept in one place. Every later change (approvals, hooks, compaction, sessions, subagents, slash commands) plugs into that loop.

## What Changes

- Add the **agent loop**: send the conversation to the model, stream its text, run every tool call it requests through the tool registry, give the results back, and repeat until the model answers without tool calls or a configurable **turn limit** is reached.
- Add a **context manager** that owns the conversation: the system prompt, the ordered history of user, assistant and tool-result messages, and a **token count** (the provider's reported usage when known, an estimate otherwise) for RRG-70 to build on.
- Add the agent's **system prompt** (based on the one in the course repository): who `nar` is, how to use tools, the working directory, platform and date, followed by the project's `AGENTS.md` when one was found.
- **Single mode** (`nar "<prompt>"`) runs the loop once; **interactive mode** (`nar`) runs it for each line and keeps the conversation between prompts. The answer is streamed with Rich; tool activity uses RRG-62's display. Ctrl+C during a turn cancels that turn and returns to the prompt.
- Tools that require approval (none of the built-in ones today) are **not run** until RRG-69 adds approvals, unless the approval level is `yolo`; the model is told why.
- Provider failures in interactive mode are reported and the session continues; in single mode they still end `nar` with exit code 1.
- New setting `max_turns` (default 20).

Out of scope: approval prompts (RRG-69), compaction and the context-usage indicator (RRG-70), loop detection (RRG-71), hooks (RRG-72), sessions (RRG-76), slash commands and prompt_toolkit input (RRG-78), new tools (RRG-64..67).

## Capabilities

### New Capabilities

- `agent-loop`: how a request is turned into model calls and tool calls, when the loop stops (final answer, turn limit, cancellation, provider failure), how tools that need approval are handled before RRG-69, and how the agent's activity is shown.
- `agent-context`: the conversation the agent keeps (system prompt contents, history across prompts, consistency after a cancelled or failed turn) and its token count.

### Modified Capabilities

- `cli-entrypoint`: single mode runs the agent instead of a one-shot answer; interactive mode runs the agent and keeps the conversation; Ctrl+C during a turn cancels the turn instead of ending the session.
- `configuration`: new `max_turns` setting.

## Impact

- **Code (`nar` repo):** new `src/nar/agent/` (loop, events, system prompt), new `src/nar/context/manager.py` (`ContextManager`), new `src/nar/ui/agent_view.py` (streamed answer, spinner, warnings), `cli/main.py` and `cli/repl.py` rewired to the agent, `config/models.py` gains `max_turns`, README updated.
- **Dependencies:** none new (`rich` came with RRG-62).
- **Interfaces other changes rely on:** the loop's event stream and its approval hook (RRG-69), the context manager's history and token count (RRG-70, RRG-76), the per-tool-call seam (RRG-71, RRG-72), building an agent with its own context and tool subset (RRG-73), the REPL's command entry point (RRG-78).
- **User-visible:** `nar "<prompt>"` and `nar` now act as a real agent that reads files; tool activity appears on standard error so standard output holds only the answer.
