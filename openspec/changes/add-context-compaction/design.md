# Design

## Context

`ContextManager` (RRG-63, `src/nar/context/manager.py`) keeps the history and `token_count()`: the usage the provider reported for the last call plus a 4-characters-per-token estimate of what came after it. `Agent.run` calls the model in a loop and uses `mark()`/`rollback()`, which are list positions, to remove a cancelled request. Tool output is already capped at 30,000 characters per call (`tools/limits.py`). `nar.cli.slash` has the `register_command` table that RRG-78 will keep.

Constraints:
- The history must stay valid for every adapter: each tool call needs its result, and a tool result needs the call before it.
- Compacting changes earlier messages, so list positions stop being a valid way to undo a request.
- The loop never prints (it yields events), and the answer goes to standard output while everything else goes to standard error.

## Goals / Non-Goals

**Goals:**
- A session keeps working past the window without the user doing anything.
- The cheap step (pruning) runs first; the model is called only when needed.
- Cancellation keeps its promise: no trace of the request.

**Non-Goals:**
- Exact token counts, model window lookup, or retrying a call the provider rejected as too long. The margin between the threshold and the window absorbs estimate errors.
- Persisting the compacted history (RRG-76).

## Decisions

### 1. Layout
```
src/nar/context/manager.py      MODIFIED  snapshots for mark/rollback, replace_messages, overhead_tokens
src/nar/context/pruning.py      NEW       prune_tool_results(), shorten_unseen_results(), unseen_start()
src/nar/context/compaction.py   NEW       ContextLimits, Compactor, render_transcript(), CompactionError
src/nar/agent/events.py         MODIFIED  ContextPruned, CompactionStarted, ContextCompacted, CompactionFailed
src/nar/agent/loop.py           MODIFIED  Agent(compactor=...), _manage_context() before each model call
src/nar/ui/agent_view.py        MODIFIED  shows the new events; context_usage()
src/nar/cli/runner.py           MODIFIED  builds the Compactor, compact_now()
src/nar/cli/repl.py             MODIFIED  indicator after each request
src/nar/cli/slash.py            MODIFIED  /compact; SlashContext gains agent and view
src/nar/config/models.py        MODIFIED  context_window, context_threshold, DEFAULT_CONTEXT_WINDOWS
```
`compaction.py` is not re-exported from `nar.context`, for the same reason as `manager.py`: it imports provider types, and the CLI loads `nar.context` on every start.

### 2. Configuration
| Key | Type | Default |
|---|---|---|
| `context_window` | int, at least 1000 | by provider: `anthropic` 200000, `openrouter` 128000, `deepseek` 128000, `ollama` 4096 |
| `context_threshold` | int, 1 to 100 (percent) | 80 |

`context_window` follows the same pattern as `model`: `None` in the field, resolved by the after-validator from `DEFAULT_CONTEXT_WINDOWS`. The defaults are conservative because nar cannot know the model: `openrouter/auto` picks one per call, and Ollama's real context depends on how the server was started (`ollama ps` showed 4096 here, which is why its default is 4096, a size at which compaction runs often because the system prompt and tools take about 2,400 tokens). **Alternative rejected:** a table per model name, which goes stale and misses most OpenRouter and Ollama models.

### 3. `ContextManager`
- `mark()` returns a frozen `ContextSnapshot` (tuple of messages, reported tokens and position) and `rollback(snapshot)` restores it. Messages are immutable, so the copy only costs the tuple. **Why:** after a compaction a list position means nothing; a snapshot also undoes the compaction itself, as the spec asks. **Alternative rejected:** forbidding rollback after a compaction, which would keep half a cancelled request.
- `replace_messages(messages)` swaps the history and forgets the reported usage, because it described the old history.
- `overhead_tokens` (default 0) is the estimate of what the history does not show, the tool definitions. It is added only when the count is a full estimate; reported usage already includes them. The agent sets it before checking the limit, since MCP tools can make the definitions large.

### 4. Pruning
`prune_tool_results(messages, keep_tokens) -> (messages, count)` is a pure function. It walks from the newest message to the oldest, adding up tool results. Results after the last assistant message are never pruned: the model has not seen them, and removing them would only make it call the tool again (the first manual run did exactly that, reading the same file three times). They are also protected from compaction, and `shorten_unseen_results` cuts them to their share of `window // 2`, with a note, when they cannot fit. **Why cut and not reject:** without the note the second manual run read a file 14 times in a row, paying for a summary each time. Then, results inside the `keep_tokens` budget stay; older ones longer than 200 characters become `[result of <tool> removed to save context; call the tool again if you need it]`. `is_error` and `tool_call_id` are kept, so the pairs stay valid. `keep_tokens` is `window // 4`. Results under 200 characters are skipped because the marker would save nothing, which also makes pruning idempotent. Tool call arguments are not pruned: the issue limits pruning to results, and large `write_file` arguments are handled by compaction.

### 5. Compaction
`Compactor(provider, limits)`:
- `over_limit(context)`: `context.token_count() > limits.limit`.
- `worth_compacting(context, current_request=None)`: whether the older part, not counting the current request, reaches `window // 20`.
- `compact(context, current_request=None)`: splits, summarizes, replaces. Raises `CompactionError` when there is nothing to compact, on an empty summary, or when the result is not smaller than the conversation; `ProviderError` propagates.

**Split:** `keep = min(window // 4, history_tokens // 2)`, where `history_tokens` is the estimate of the messages alone: the system prompt and the tool definitions are not reduced by a summary, and counting them would leave a short conversation with nothing to summarize. Walking from the end, the tail starts at the earliest message that is not a tool result and whose tail fits in `keep`, but always early enough to include unseen results and the call they answer. The tail can be empty. A tail may start with an assistant message (it follows the summary, a user message) or with a user message; two user messages in a row are accepted by both APIs. Keeping at most half of the history means that `/compact` frees space at any size.

**Summary call:** `provider.complete([Message.user(prompt)], system=SUMMARY_SYSTEM, tools=(), max_tokens=min(4096, window // 4))`. The older messages are rendered as a text transcript instead of being sent as messages: the summary call has no tools, and the output must not continue the conversation. In the transcript, a tool result is cut to 2,000 characters and tool arguments to 500. When the whole transcript is over 70% of the window, its middle is replaced by a marker, keeping the first quarter (where an earlier summary is) and the end. The system prompt asks for the user's goals, decisions, files and changes, errors, and what is still pending, and says the transcript is data.

**Result:** one user message with the header from the spec, the summary, and, when `current_request` is not in the tail, `The user's current request:` with its text. The header matters because a summary in a user message has more authority than the tool output it came from: a prompt-injected page should not become a user instruction by being summarized.

**Why a compaction is skipped under 5%:** when the system prompt or one huge request is what fills the context, compaction would run on every turn and pay for a summary that frees nothing. The current request does not count, because its text stays word for word. The agent checks `worth_compacting`; `/compact` does not. A result that is not smaller is discarded for the same reason.

### 6. The loop
`Agent(..., compactor=None)`: without a compactor nothing changes, so the existing tests and any later embedding stay as they are. `_manage_context(prompt)` runs before each `ModelCallStarted`:

1. Set `overhead_tokens`; return if not over the limit.
2. Prune, and shorten unseen results; yield `ContextPruned(count, before, after, shortened)` if something changed; return if no longer over.
3. Return silently if not `worth_compacting`, because a warning would repeat on every turn. This is checked before `CompactionStarted`, so the spinner never shows a compaction that does not happen.
4. Yield `CompactionStarted`; call `compact`. On success yield `ContextCompacted(before, after)`. A `ProviderError` or a `CompactionError` becomes `CompactionFailed(message)`.

`KeyboardInterrupt` propagates and the existing `finally` restores the snapshot. Since `/compact` runs outside `Agent.run`, it does not need a snapshot: `replace_messages` happens only after the summary arrived.

### 7. Display
`AgentView` shows: `context: removed N old tool results (A -> B tokens)` as a note; a spinner `compacting the conversation...`; `context: compacted (A -> B tokens)` as a note; `warning: could not compact the conversation: <reason>; continuing with the full history`. `context_usage(tokens, window)` prints `context: 12,300 / 128,000 tokens (10%)` as a note, from `repl.run`, on standard error. It is not an event because single mode must not show it and it is not part of the loop.

### 8. `/compact`
`SlashContext` gains optional `agent` and `view` (default `None`, so the `/mcp` tests stay valid). The command calls `runner.compact_now(agent, view)`, imported inside the handler because the runner pulls in the SDKs. `compact_now` catches `KeyboardInterrupt`, `ProviderError` and `CompactionError`, and shows them through the view, since `repl.run` dispatches commands outside its request `try`.

## Risks / Trade-offs

- **The estimate is rough** (4 characters per token, worse for code or CJK) → the 80% threshold leaves margin, and reported usage corrects it after every call.
- **A summary loses detail** → pruning runs first, the latest messages stay complete, and the model can read files again.
- **A reasoning model may spend the summary's `max_tokens` on thinking** (seen with `qwen3.5` in RRG-61) → it arrives as an empty summary, which warns and keeps the history.
- **Ollama drops text silently beyond its real context** → the default window is small, and the README says to raise it only together with the server's context length.
- **Files changed by tools are not undone on cancel** → same as today; only the conversation is restored.

## Migration Plan

Additive. The new settings have defaults. Sessions are not persisted yet, so there is no stored history to convert.

## Open Questions

None.
