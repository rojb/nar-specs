# Design

## Context

`Agent` (`src/nar/agent/loop.py`) takes a provider, a registry and a `ContextManager`, plus optional `gate`, `compactor`, `detector` and `hooks`. It yields events and never prints. `Agent.run` rolls back on anything but completion. `ToolRegistry.execute` catches `Exception` from a tool, so `KeyboardInterrupt` passes through. Tool output is capped at 30,000 characters. `connect_mcp` adds MCP tools after `create_agent`, and custom tools load inside it. The research behind these decisions is in `docs/SUBAGENTES.md`.

Constraints:
- A subagent must be unable to escape approvals or hooks.
- `Tool.execute` is synchronous and returns one result, while a subagent's work is a stream of events.
- The spinner and the approval prompt share standard error.

## Goals / Non-Goals

**Goals:**
- Reuse `Agent` unchanged, so limits, compaction and loop detection come for free.
- Make "read only" a property of the registry, not a request to the model.

**Non-Goals:**
- Parallel or background subagents, user-defined types, per-subagent models, saved transcripts.

## Decisions

### 1. Layout
```
src/nar/agent/subagent.py   NEW       SubagentType, TaskTool, SubagentProgress
src/nar/agent/prompt.py     MODIFIED  guideline for task; build_subagent_prompt()
src/nar/tools/registry.py   MODIFIED  subset(names)
src/nar/config/models.py    MODIFIED  subagent_max_turns
src/nar/cli/runner.py       MODIFIED  create_agent registers TaskTool
src/nar/ui/agent_view.py    MODIFIED  subagent_progress(), stop_waiting_for_question()
```

### 2. Configuration
| Key | Type | Default |
|---|---|---|
| `subagent_max_turns` | int, at least 1 | 15 |

### 3. `TaskTool`
Schema: `subagent` (enum `explorer`, `general`), `prompt` (string, `minLength` 1), both required, `additionalProperties: false`. The registry's validation produces the error for an unknown type. `requires_approval = False`, because launching changes nothing; each call inside goes through the gate.

It receives what it needs in its constructor: the parent `Agent`'s provider, registry, gate and hooks, the limits, `subagent_max_turns`, the system prompts, and a `progress` callback. `execute` creates, for each call, a new `ContextManager`, a new `LoopDetector`, a new `Compactor` and a new `Agent`, then consumes `agent.run(prompt)` inside `contextlib.closing`.

**Why a new `Agent` and not a parameter on the loop:** isolation then holds by construction, since nothing in the subagent has a reference to the parent's `ContextManager`.

### 4. Tools of each type
`ToolRegistry.subset(names)` returns a new registry with the same context and the same `Tool` instances. Computed on each call, because MCP tools arrive after `create_agent`.
- `explorer`: the fixed tuple `read_file`, `grep`, `glob`, `list_dir`.
- `general`: all names of the parent, minus `task` and `todos`.

Not having `task` is what limits depth, so no counter is needed. **Why not offer `todos` to `general`:** the one instance holds the main agent's list, and a subagent would overwrite it.

### 5. Prompts
`build_subagent_prompt(role, cwd, instructions)`: identity, role, environment, and project instructions when present. Saved memory is left out: it is about the user and the conversation, which a subagent does not have. It does not reuse `build_system_prompt`, because the guidelines mention `todos` and `memory`. Both roles end with: the last message is the only thing the main agent receives, so it must stand alone and include file paths.

The main system prompt gains one guideline: use `task` with `explorer` for broad searches of the code; do not use it to read a known file; give it a complete prompt because it cannot see the conversation.

### 6. Results
| Subagent ended with | Result |
|---|---|
| `RequestFinished` with text | `ok(text)` |
| `TurnLimitReached` | `ok(last text + note that the work may be incomplete)`; an error if there is no text at all |
| `LoopAborted` | same, with a note saying it repeated calls |
| `RequestFinished` with no text | `error("the subagent returned no answer")` |
| `ProviderError` | `error(message)` |
| `KeyboardInterrupt` | propagates, and the parent's loop rolls back |

`summary` is `explorer: 12 tool calls, 3 turns`, shown by the usual result line. Failures use `ToolError`-style error results, so the failure envelope of RRG-71 applies. **Why "may be incomplete" is `ok`:** the partial text is still useful; the note is what stops the main agent from trusting it as complete.

### 7. Approvals and hooks
The parent's `gate` object is passed as it is, so level and "always" answers are shared, including a change made with `/approval` (RRG-78). The hooks object is passed as well, so `before_tool` can block a subagent's call. `before_agent` and `after_agent` then also fire for each subagent, with its prompt; the README mentions this.

Spinner: `TaskTool` wraps the gate so that, for a tool that requires approval, `progress.pause()` stops the waiting line before the question. The prompt names the tool; the preceding `task` call line shows who is asking.

### 8. Display
`SubagentProgress` is a small protocol with `update(kind, tool_calls)` and `pause()`, implemented by `AgentView`. Each `ToolCallStarted` updates one status line on standard error (`explorer: 3 tool calls...`). `TextDelta` events are dropped; the answer returns only as the result. On a non-terminal, `rich` status prints nothing, which keeps pipes clean. Retry notices keep working through `view.retry`.

**Why no expandable view:** `nar` prints a stream, with no full-screen TUI that could unfold a row. Here "folded" means the activity is summarized in one line.

### 9. Slices
PR A: groups 2 and 3 of `tasks.md`, the tool with no CLI wiring. PR B: groups 4 and 5, wiring, display, guidance and README.

## Risks / Trade-offs

- **More tokens overall** → each subagent makes its own calls. Documented in the README, and the guideline limits it to broad searches.
- **A weak summary loses information** → the role prompt asks for paths and a self-contained answer, and the main agent can read the files or call `task` again.
- **The model may not delegate** → the done-when depends on the model's choice. The guideline and description push for it, and the manual check records what happened.
- **A prompt-injected file reaches the subagent** → an `explorer` cannot write, and a `general` still asks. Its summary can carry the injected text back as a tool result, the same level of trust as any tool output.
- **Hooks fire more often** → `before_agent` and `after_agent` run per subagent as well. Accepted and documented, because skipping `before_tool` would be a way around a block.
- **Long silent waits in a pipe** → no output until the result. Same as a slow model call today.

## Migration Plan

Additive. The new setting has a default.

## Open Questions

None.
