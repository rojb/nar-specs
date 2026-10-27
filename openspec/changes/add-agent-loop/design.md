# Design

## Context

After RRG-61 and RRG-62, `nar` has a synchronous `LLMProvider` (`stream()` yields `TextDelta` pieces and ends with `ResponseDone(LLMResponse)`), internal `Message`/`ToolCall` types, and a `ToolRegistry` whose `execute()` never raises for model mistakes and whose `definitions()` export the neutral shape the providers accept. RRG-62's design fixed the seam this change uses: the *caller* of `registry.execute` wraps it with `ToolRenderer.show_call`/`show_result`, and a result becomes `Message.tool_result(call.id, result.output, is_error=result.is_error)`. Single mode currently streams one answer with `click.echo`; interactive mode is RRG-60's `input()` placeholder. Motivation: see `proposal.md`. Behavior: see `specs/agent-loop`, `specs/agent-context`, `specs/cli-entrypoint`, `specs/configuration`.

Constraints:
- The provider SDKs take about 2 s to import (RRG-61 Decision 8), so `nar --version` and `--show-config` must keep importing them lazily.
- Windows PowerShell and Linux/macOS; Rich already handles both.
- `add-llm-providers` (RRG-61) is merged but not archived. Its `cli-entrypoint` delta modifies "Single mode"; this change's delta for the same requirement already contains RRG-61's scenarios, so **`add-llm-providers` must be archived before `add-agent-loop`**.

## Goals / Non-Goals

**Goals:**
- A loop with no UI inside, so subagents (RRG-73) and tests drive it without a terminal.
- One place that owns the conversation and its size (RRG-70 and RRG-76 build on it).
- Seams for approvals (RRG-69), loop detection (RRG-71) and hooks (RRG-72) that do not require rewriting the loop.

**Non-Goals:**
- Markdown rendering of answers. Text is streamed as-is (Decision 6).
- Async or parallel tool execution: tools run sequentially, as the spec requires.
- prompt_toolkit input, history and slash commands (RRG-78). The REPL keeps `input()`.

## Decisions

### 1. Layout
```
src/nar/
├── agent/                    NEW
│   ├── __init__.py           NEW  re-exports Agent, events, build_system_prompt
│   ├── events.py             NEW  agent events (Decision 2)
│   ├── loop.py               NEW  Agent.run(), ToolGate, approval_gate()
│   └── prompt.py             NEW  build_system_prompt()
├── context/
│   └── manager.py            NEW  ContextManager
├── ui/
│   └── agent_view.py         NEW  AgentView: streamed text, spinner, tool lines, warnings
├── cli/
│   ├── main.py               MODIFIED  single mode runs the agent; Ctrl+C → exit 130
│   ├── repl.py               MODIFIED  each line runs the agent; Ctrl+C cancels the request
│   └── runner.py             NEW  create_agent(), run_request()
└── config/models.py          MODIFIED  max_turns
```

### 2. The loop is a generator of events
`Agent(provider, registry, context, *, max_turns, gate)`; `Agent.run(prompt) -> Iterator[AgentEvent]`. Events (frozen dataclasses in `agent/events.py`):

| Event | When |
|---|---|
| `ModelCallStarted(turn)` | before each model call |
| `TextDelta(text)` (from `nar.providers`) | each streamed piece of text |
| `ToolCallStarted(call)` / `ToolCallFinished(call, result)` | around each tool call |
| `AnswerCutOff(model)` | a model answer stopped at `max_tokens` |
| `TurnLimitReached(max_turns)` | the limit stopped the request |
| `RequestFinished(response)` | final answer with no tool calls |

Per turn: `provider.stream(context.messages, system=context.system_prompt, tools=registry.definitions())`; text deltas are re-yielded; the final `LLMResponse` is added to the context; no tool calls → `RequestFinished`; otherwise each call goes through `gate` and `registry.execute` in order and its result is added with `context.add_tool_result`. If the answer of turn `max_turns` still has tool calls, they are **not** run: each gets an error result `not run: the turn limit (N) was reached` (spec: every tool call in the history has a result) and `TurnLimitReached` ends the request.
- **Why a generator:** the CLI renders events, tests collect them, RRG-73 can drain them silently, RRG-71 can watch `ToolCallStarted`. **Alternative rejected:** callbacks (`on_text`, `on_tool_call`, …) — more parameters to thread through and harder to cancel.

### 3. Cancellation and failures roll back the request
`run()` takes `mark = context.mark()` before adding the user message and wraps its body in `try/finally`: unless the request completed (final answer or turn limit), `context.rollback(mark)`. This covers every exit path: `ProviderError` from the provider, `KeyboardInterrupt` raised inside the generator, and the consumer closing the generator after a `KeyboardInterrupt` raised in its own rendering code (`GeneratorExit`). The CLI always iterates inside `contextlib.closing(...)` so the generator is closed at once. A cancelled or failed request therefore leaves no trace (spec), and no history ever ends with unanswered tool calls.

### 4. `ContextManager` (`context/manager.py`)
```python
ContextManager(system_prompt: str)
  .system_prompt -> str
  .messages -> tuple[Message, ...]
  .add_user(text) / .add_assistant(response: LLMResponse) / .add_tool_result(call_id, result: ToolResult)
  .mark() -> int / .rollback(mark)
  .token_count() -> int
```
`add_assistant` stores `response.to_message()` and, when `response.usage` is present, remembers `input_tokens + output_tokens` and the history length at that point. `token_count()` = that reported number + an estimate of the messages added since; without usage, an estimate of system prompt + all messages. Estimate: `ceil(characters / 4)` over text, tool-call names and JSON arguments. A rollback below the remembered point forgets the reported usage.
- **Why estimate instead of `tiktoken`/`count_tokens`:** each provider tokenizes differently, the Anthropic count is an extra HTTP call, and RRG-70 only needs a threshold signal; the provider's own number replaces the estimate after every call. **No new dependency.**

### 5. System prompt (`agent/prompt.py`)
`build_system_prompt(cwd, instructions, *, platform_name, today) -> str`. Sections, in English: identity (`nar`, a coding agent in the user's terminal); how to work (read files before answering questions about them, never invent file contents or tool results, prefer small focused tool calls, answer concisely and reference paths relative to the working directory); environment (working directory, OS from `platform.system()`/`platform.release()`, date); then, when `AGENTS.md` was loaded, `# Project instructions (from <path>)` followed by its content. Based on the structure of the course repository's prompt, rewritten for `nar`'s tools. Built once per process; the date does not change mid-session in practice.

### 6. Terminal view (`ui/agent_view.py`)
`AgentView(out: Console, err: Console)` consumes events:
- Text → `out.print(text, end="", markup=False, highlight=False, soft_wrap=True)`; remembers whether the last character was a newline so the answer always ends with one.
- `ModelCallStarted` → `err.status(f"waiting for {model}...")` (transient spinner); stopped on the first text or tool event and in a `finally`.
- Tool events → RRG-62's `ToolRenderer(err)`. Before the first tool line, an unfinished text line on stdout is closed with a newline so the terminal stays readable.
- `AnswerCutOff`, `TurnLimitReached`, retry notices, cancellation and errors → yellow/red one-liners on `err`.
- Rich detects non-terminals: when stderr is redirected, the spinner prints nothing.
- **Why stdout for text only:** `nar "..." > file` gives just the answer (spec). **Alternative rejected:** `rich.live` re-rendering Markdown while streaming — it repaints the whole answer on every chunk, flickers on long answers and cannot be piped.
- This replaces RRG-61's `click.echo`/`click.secho` path in single mode; `click.secho` stays only in the CLI error boundary.

### 7. Approval before RRG-69: `ToolGate`
`ToolGate = Callable[[Tool, Mapping[str, Any]], ToolResult | None]`: returning a result skips the tool and sends that result to the model. `approval_gate(level)` returns the gate for this change: when `tool.requires_approval` and `level` is not `yolo`, return `ToolResult.error("<tool> needs user approval, which nar cannot ask for yet; ...")`. Unknown tool names skip the gate and let `registry.execute` report them. RRG-69 replaces `approval_gate` with one that asks the user; RRG-72 can compose its `before_tool` hook with the same signature.

### 8. CLI wiring (`cli/runner.py`)
`create_agent(config, cwd, instructions, view) -> Agent` builds the provider (`create_provider(config, on_retry=view.retry)`), `create_default_registry(cwd)`, `ContextManager(build_system_prompt(...))` and `approval_gate(config.approval)`. `run_request(agent, view, prompt) -> Outcome` (`FINISHED`, `TURN_LIMIT`) iterates `closing(agent.run(prompt))` and feeds the view. `runner.py` is imported inside the functions that need it, keeping the SDK import off the fast commands.
- **Single mode:** `run_request`; `TURN_LIMIT` → exit code 1. `ProviderError` propagates to the existing error boundary (exit 1, nothing on stdout). Ctrl+C reaches `main()` as Click's `Abort`; `main()` now prints `cancelled` and exits with 130 (the shell convention for SIGINT) instead of `Aborted!` and 1.
- **Interactive mode:** `repl.run(agent, view, instructions)`; per line, `KeyboardInterrupt` → `view.cancelled()`, `ProviderError` → `view.error(message)`, then the prompt returns. Ctrl+C or EOF *at the prompt* still end the session with 0 (unchanged). The agent (and its context) is created once, before the first prompt, so a missing API key fails at startup instead of after the first question.

### 9. Configuration
`max_turns: int = Field(default=20, ge=1)` in `NarConfig`. The loader's existing translation reports the setting, file and pydantic's message (`greater than or equal to 1`). No other new keys.

### 10. Tests
- `tests/test_agent_loop.py`: a scripted fake provider (list of `LLMResponse`s, records the messages it received) and a registry with test tools — final answer, two calls in order, error results, turn limit (results added for unrun calls), approval gate at `ask`/`yolo`, rollback on `ProviderError` and on closing the generator mid-stream.
- `tests/test_context_manager.py`: history order, mark/rollback, token count with and without usage.
- `tests/test_system_prompt.py`, `tests/test_agent_view.py` (Rich consoles on `StringIO`), and CLI tests with the fake provider for both modes, exit codes 0/1/130 and stdout/stderr separation.

## Risks / Trade-offs

- [Small local models ignore tools or loop until the limit] → `max_turns` bounds the cost; RRG-71 adds loop detection. The "done when" is checked with OpenRouter and a local model.
- [A `KeyboardInterrupt` inside a tool's file I/O could leave a partial effect] → Only `read_file` exists (read-only); write tools (RRG-64) and approvals (RRG-69) must keep their own writes atomic.
- [The estimate is off by up to ~30 % for code-heavy text] → It is replaced by the provider's number after every call; RRG-70 sets thresholds with margin.
- [System prompt wording affects every answer] → Kept in one function with a test on its sections, so tuning it later is a one-file change.
- [Archive order with `add-llm-providers`] → Stated above and in tasks; `openspec archive` of this change before RRG-61's would drop RRG-61's provider scenarios from "Single mode" only if RRG-61's delta were archived afterwards — archiving in order avoids it.

## Migration Plan

No migration: no new dependency, one new optional setting. `nar` and `nar "<prompt>"` now behave as an agent. Rollback: revert the PR; single mode returns to RRG-61's one-shot answer.
