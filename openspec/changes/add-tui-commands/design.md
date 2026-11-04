# Design

## Context

`nar.cli.repl.run` reads a line with `input(PROMPT)`, treats `exit` as the way out, and hands every
other line to `nar.cli.slash.dispatch`, which looks the name up in `_COMMANDS` — a table filled by
`register_command(name, handler, summary)`. RRG-74 built that seam and RRG-70/76 added four commands to
it. `SlashContext` already carries what a command needs: the MCP manager, the two consoles, the agent,
the view and the recorder.

The agent exposes its provider (`agent.provider`, with `name` and `model`), its registry
(`agent.registry.names()`, `definitions()`), its gate (`agent.gate`, an `ApprovalGate` built by
`approval_gate(level, prompter, context)`) and its conversation (`agent.context`). `create_agent`
builds the provider through `factory.create_provider(config)`, which is where a missing key fails, and
`runner.show_context_usage(agent, view)` already prints the tokens line. So every base command is a
read or a replacement of something the agent already holds; none of them needs new machinery.

The constraint that decides the shape: the REPL tests drive standard input through `click`'s
`CliRunner`, and `nar` is used in scripts (`printf '/help\n' | nar`). A prompt that always went through
`prompt_toolkit` would break both.

## Goals / Non-Goals

**Goals:**
- Every registered command is discoverable and completable.
- Provider, model and approval level can change without losing the conversation.
- The interactive prompt is the only place that knows about `prompt_toolkit`.

**Non-Goals:**
- A configuration editor, per-request provider flags from the prompt, path completion, custom key
  bindings, history search beyond the default, and rewriting the provider recorded in an existing
  session file.

## Decisions

### 1. Layout
```
src/nar/cli/prompt.py       NEW       read_line(), SlashCompleter, prompt_session()
src/nar/cli/commands.py     NEW       the eight base commands, registered at import
src/nar/cli/repl.py         MODIFIED  uses read_line() instead of input()
src/nar/cli/slash.py        MODIFIED  expose the table (commands()) for /help and the completer
src/nar/cli/runner.py       MODIFIED  switch_model() and clear_conversation()
pyproject.toml              MODIFIED  prompt-toolkit dependency
README.md                   MODIFIED  the command table
```

### 2. `prompt-toolkit` only where it helps
`read_line(prompt)` returns `input(prompt)` when `sys.stdin.isatty()` is false, and otherwise builds a
`PromptSession` with a `Completer` over the registered command names. The import lives inside that
branch, so a pipe, a test run, `nar "..."` and `--help` never load the library. **Why:** completion is
a terminal affordance, not a dependency of the program; and the alternative — always using the
library — would need `dummy_input` plumbing in every test for no behavior anybody sees. The completer
completes only the first word when it starts with `/`, offers `/name` with the registered summary as
its display meta, and leaves the rest of the line alone.

### 3. The base commands
`commands.py` registers them at import; `repl.run` imports it next to `slash`, so the table is complete
before the first line is read. Each handler gets `(context, argument)` like the existing ones.

| Command | Behavior |
|---|---|
| `/help` | every `slash.commands()` entry as `/name  summary`, alphabetical, on standard error |
| `/clear` | `runner.clear_conversation(agent, recorder)`: replace the messages with none, keep the system prompt, save the session, note that the earlier messages are gone |
| `/exit` | raise `SystemExit(0)` after `view.finish()`; the REPL's loop is the same one `exit` uses, so `/exit` ends the session |
| `/model` | no argument → provider, model and source; with argument → `runner.switch_model(agent, view, argument)` |
| `/approval` | no argument → the level; with argument → set `agent.gate.level` after validation |
| `/config` | the `LoadedConfig.describe(cwd)` lines, which already mask secrets |
| `/tools` | `agent.registry` names with the first line of each description |
| `/tokens` | `runner.show_context_usage(agent, view)` |

`/model` accepts `provider`, `provider/model` and `provider model`. **Why those three:** the first is
the common case and matches `--provider`; the other two mirror what people type after seeing the
`provider/model` pair in `/model`'s own output. A missing credential is caught by
`factory.create_provider` raising `NarError` before anything is replaced, so the switch is atomic: the
new provider is built first and assigned only if it worked.

The conversation, the registry and the session are untouched by `/model`; `view.waiting_text` is
updated so the spinner names the new model. The session file keeps the provider it was created with —
recorded as out of scope, and the README says so.

### 4. `/approval` needs the level, not a new gate
`ApprovalGate` holds `level`; the session's "always" answers live in the same object, so changing
`level` in place is enough and no tool is rebuilt. The level is parsed with `ApprovalLevel(value)`, and
a bad value lists `ask`, `auto-edit`, `yolo`.

### 5. Errors never end the session
Every handler reports through `context.err` and returns; `repl.run` already calls `dispatch` outside
its request `try`, so a handler must not raise. The one exception is `/exit`, which is the point.

## Risks / Trade-offs

- **One new runtime dependency** → imported only in the terminal branch, and it is already in the
  project's stated stack; `pyproject` pins a minimum, and `pip install -e .` brings it.
- **A handler that raises would kill the session** → each handler is written to catch and report; the
  `/model` path is the only one that runs code that can fail (building a provider).
- **`/clear` throws away context the user may want** → it says so explicitly, and the session file is
  rewritten so a later `--resume` does not resurrect the discarded messages.
- **Completion is names only** → enough for eight commands and the four other changes add; arguments
  are documented in the summaries instead.

## Migration Plan

Additive. Without a terminal nothing changes; the four existing commands keep their behavior.

## Open Questions

None.
