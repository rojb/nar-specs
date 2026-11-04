# Design

## Context

`Agent.run` (`src/nar/agent/loop.py`) runs one request: a model call, the tool calls it asks for, their
results, and another model call, up to `max_turns` (default 20). It never prints: it yields events
that `AgentView` renders, the answer goes to standard output by `TextDelta`, and everything else goes
to standard error. `Agent(gate=...)` already intercepts a tool call before it runs (RRG-69) and
returns a `ToolResult` instead, so the loop has one place where a call can be skipped.

`ToolRegistry.execute` (`src/nar/tools/registry.py`) is where `nar` turns its own failures into a
`ToolResult.error(...)`: an unknown tool, arguments that do not match the schema, a `ToolError` and
an unexpected exception. The message is what the model sees; the OpenAI-compatible adapter adds an
`Error: ` prefix and the Anthropic adapter sets `is_error`. Nothing records which of those four
failures it was, so nothing can tell the model anything about the failure beyond its text.

The context management added by RRG-70 was built against exactly this problem: its design records
manual runs where the agent read the same file three times in a row, and another where it read it 14
times, paying for a compaction each time.

Constraints:
- The history must stay valid for every adapter: every tool call keeps its result, and the request
  ends either with an assistant message or right after the results of the calls it ran.
- The loop must keep working without a detector: `Agent(...)` is built in tests and by embeddings,
  and RRG-63's behavior must not change when the detector is off.
- A cancellation must still remove the whole request, including the messages this change adds.

## Goals / Non-Goals

**Goals:**
- Stop a repeating agent early and cheaply, without an extra model call and without losing the work
  already done.
- Tell the model what is wrong in a way it can act on, before giving up on the request.
- Keep the change additive: no new configuration key, no MODIFIED requirement in the spec store.

**Non-Goals:**
- Judging the model with a second model, or learning thresholds per model.
- Detecting repetition in the model's prose, or across requests and sessions (RRG-76 stores them).
- Undoing side effects: a stopped request keeps the files its tools changed, as any request does.
- Making the detector configurable. If it ever needs a switch, it is a later change with its own
  spec; the thresholds are constants in one module today.

## Decisions

### 1. Layout
```
src/nar/agent/loops.py       NEW       LoopDetector, FailureKind use, canonical_arguments(), render_failure()
src/nar/tools/base.py        MODIFIED  ToolResult.error_kind, FailureKind
src/nar/tools/registry.py    MODIFIED  sets the failure kind on the four failures it produces
src/nar/agent/approval.py    MODIFIED  the rejection is marked FailureKind.REFUSED, never enveloped
src/nar/agent/events.py      MODIFIED  LoopDetected, LoopAborted
src/nar/agent/loop.py        MODIFIED  Agent(detector=...), detection after each turn, envelope for the model
src/nar/ui/agent_view.py     MODIFIED  shows the two new events on standard error
src/nar/cli/runner.py        MODIFIED  Outcome.LOOP
src/nar/cli/main.py          MODIFIED  a stopped request exits non-zero in single mode
```
`loops.py` imports nothing from `nar.agent`, so `nar/agent/__init__.py` stays a plain package and the
detector can be unit-tested without a provider.

### 2. Configuration
**No new keys.** The three thresholds are module constants in `loops.py`: `REPEAT_LIMIT = 3`,
`FAILURE_REPEAT_LIMIT = 2`, `OSCILLATION_CYCLES = 2`. **Why:** the issue asks for detection, not for
tuning, and every setting is a promise the README has to keep; a wrong default is fixed by a
constant, a wrong setting needs a user to find it.

### 3. What counts as a repetition
`canonical_arguments(arguments)` renders the arguments as JSON with sorted keys, no spaces and
`ensure_ascii=False`; anything that cannot be serialized falls back to `repr(sorted(items))`. The
signature is `(call.name, canonical_arguments(call.arguments))`, so `{"a": 1, "b": 2}` and
`{"b": 2, "a": 1}` are the same call and `read_file` on two paths is not.

`LoopDetector` keeps a list of `ExecutedCall(name, signature, failed)` for the current request and
`reset()` starts it again; `Agent.run` resets it at the beginning, which is also what makes
`/compact` and a new request independent. `check()` looks only at the tail of that list:

| Pattern | Condition | Kind |
|---|---|---|
| repeat | the last `REPEAT_LIMIT` calls share one signature | `REPEAT` |
| repeat of a failure | the last `FAILURE_REPEAT_LIMIT` calls share one signature and the last one failed | `REPEAT` |
| alternation | the last `2 × OSCILLATION_CYCLES` calls are `A, B, A, B` with `A ≠ B` | `OSCILLATION` |

**Why the tail and not the counts:** a model may legitimately call the same tool twice with the same
arguments (`read_file` after an `edit_file` that reported no change). A tail of two is the warning
case only when both calls failed, because then the model has already been told how it failed and has
still not changed anything. **Why the arguments and not the name:** `read_file` on different files is
a normal exploration, and pruning it would make the detector useless.

**Alternation needs two full cycles:** one full edit-read pair is a normal step; two in a row, with
identical arguments and nothing changing, is thrash. The false positive remains possible, and the
warning-first design is what keeps it cheap: the model usually answers after the warning.

### 4. Escalation
`LoopDetector` also holds `warned`. `check()` returns `None`, `WARN` or `STOP`:

1. no pattern → `None`.
2. pattern and `warned` is false → `WARN`, and `warned` becomes true; the caller resets the tail, so
   the same pattern has to build up again from scratch before it can stop the request.
3. pattern and `warned` is true → `STOP`.

The warning is one user-role message with a header, in the same style as the compaction summary:

```
[nar] The last tool calls repeated: shell({"command": "pytest -q"}) three times. Stop repeating
them: use the result you already have, change the arguments, or take another approach. If the same
calls repeat again, this request will be stopped.
```

**Why a user message with a header:** adapters accept a user message after tool results (RRG-70 uses
the same shape for the summary), and the header marks it as generated by `nar`, so text quoted from a
file or a tool never reads as an instruction coming from the user. **Alternative rejected:** an
assistant message, which the model can treat as its own earlier reasoning and keep following.

### 5. Structured failure feedback
`FailureKind` (`StrEnum` in `src/nar/tools/base.py`): `UNKNOWN_TOOL`, `INVALID_ARGUMENTS`,
`TOOL_ERROR`, `UNEXPECTED_ERROR`, `REFUSED`. `ToolResult` gains `error_kind: FailureKind =
FailureKind.TOOL_ERROR`, and `ToolRegistry.execute` sets it at each of its four failure paths; the
approval rejection sets `REFUSED`. `render_failure(call, result)` produces what the model receives:

```
[tool error] shell
kind: tool error
message:
pytest: command not found
Repeating this call will fail the same way. Change the arguments or the approach.
```

`REFUSED` is never wrapped: the rejection text already explains that the user said no and must not be
followed by "change the arguments", which would invite the model to look for a way around it. Every
other kind is wrapped, and the tool's own text appears unchanged inside the envelope, so the
`Error: ` prefix the OpenAI adapter adds still applies to a readable sentence.

`Agent._feedback(call, result)` returns the result as it is when it did not fail or is `REFUSED`, and
a copy with the enveloped text when it did. The loop adds the copy to the conversation and keeps
yielding the original `ToolCallFinished`, so the terminal shows what the tool said and not the
envelope.

### 6. Where it runs
In `run()`, right after the `for call in response.tool_calls` loop and before the next turn:

1. `decision = self.detector.check()`; without a detector nothing happens.
2. `STOP` → `completed = True`, `yield LoopAborted(kind, call, count)` and `return`. The history keeps
   the request, because every call has its result and the request is complete from the provider's
   point of view.
3. `WARN` → `self.context.add_user(WARNING.format(...))`, `self.detector.reset_tail()`,
   `yield LoopDetected(kind, call, count)` and continue with the next turn.

The turn limit is unchanged and is checked before the tools of the last turn run, so a request can
end for one reason only. `KeyboardInterrupt` keeps propagating through the existing `finally`, which
rolls the whole request back, including the warning message.

### 7. Display and outcome
`AgentView` prints, on standard error, in the style of the turn-limit line:
- `warning: the agent repeated shell with the same arguments 3 times; asked it to change strategy`
- `stopped: the agent kept repeating shell after the warning`

`Outcome` gains `LOOP = "loop"`; `run_request` sets it on `LoopAborted`, and `main.py` returns a
non-zero code for `TURN_LIMIT` and `LOOP` in single mode. Interactive mode returns to the prompt in
both cases.

## Risks / Trade-offs

- **A false positive stops a working request** → the warning comes first, the tail is reset after it,
  and stopping only ends the request: the files are already changed and the conversation keeps
  everything, so the user can ask again. Detection needs identical arguments, which excludes normal
  exploration.
- **A model ignores the warning** → it costs one extra model call, which is what the change is for.
- **The envelope changes what the model reads** → it is the same text inside a fixed frame, and the
  tests that assert a failing result text have to expect the frame; the terminal output does not
  change.
- **Tool names reach the terminal inside a warning line** → they are printed literally, as the `/mcp`
  view already does, so Rich markup in a name cannot break the line.
- **A detector bug could stop requests too early** → `Agent(detector=None)` is the default, the
  detector is a pure function over a list, and its unit tests do not need a provider.

## Migration Plan

Additive. No configuration key, no stored data, no change to a spec owned by another change. A session
file written by RRG-76 that contains a warning message is read back like any other message.

## Open Questions

None.
