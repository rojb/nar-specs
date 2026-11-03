# Design

## Context

`ContextManager` holds the history as immutable `Message` objects and restores it with `rollback` after a cancelled request. `Agent.run` yields `ModelCallStarted` before each model call, and `runner.run_request` consumes the events. `tools/memory.py` already writes JSON atomically under `nar_home()`. `nar.cli.slash` has `register_command`, which RRG-78 will keep.

Constraints:
- A saved history must be valid for the providers at any moment the process can die: every tool call needs its result.
- Tests must not touch the real home folder (they set `NAR_HOME`).
- `nar.cli` must stay fast to load, so `nar --help` and `--show-config` do not import the session or provider code.

## Goals / Non-Goals

**Goals:**
- A closed terminal loses at most the turn in progress.
- No change to how the agent loop behaves.
- Files that RRG-77 can extend.

**Non-Goals:**
- Safe concurrent writes to the same session, indexes, retention, restoring `todos` or approvals.

## Decisions

### 1. Layout
```
src/nar/session/__init__.py   NEW       re-exports
src/nar/session/store.py      NEW       Session, SessionStore, SessionFileError, sessions_dir()
src/nar/session/recorder.py   NEW       SessionRecorder
src/nar/ui/session_view.py    NEW       render_history(), render_sessions()
src/nar/agent/loop.py         MODIFIED  measure_tools(), so the usage line counts tool definitions after a resume
src/nar/cli/main.py           MODIFIED  --resume; run_single resumes
src/nar/cli/runner.py         MODIFIED  run_request(recorder=...), open_session(), switch_session()
src/nar/cli/repl.py           MODIFIED  creates the recorder and shows the history
src/nar/cli/slash.py          MODIFIED  /sessions, /resume; SlashContext gains recorder
```
No new configuration keys.

### 2. File
```json
{"version": 1, "id": "20261103-221500-a1b2", "created_at": "...", "updated_at": "...",
 "cwd": "/path", "provider": "openrouter", "model": "openrouter/auto", "messages": [...]}
```
A message is `{"role", "content", "tool_calls": [{"id", "name", "arguments"}], "tool_call_id", "is_error"}`. Dates are UTC ISO 8601. The id is the UTC date plus 4 random hex characters, so ids sort by creation and are short enough to type. `version` lets RRG-77 add fields; any other version is an error, not a guess.

Writes follow `MemoryStore`: `tempfile.mkstemp` in the same folder (created with mode 0600), then `os.replace`. The folder is created with mode 0700. **Why these modes:** sessions hold file contents and command output, which can include secrets.

`SessionStore(directory)`: `save`, `load(id)`, `delete(id)`, `list() -> (sessions, invalid_count)`, `find(prefix)`, `latest(cwd, exclude=None)`. Loading validates the shape and raises `SessionFileError(NarError)` naming the file. Ids must match `^[0-9A-Za-z-]+$` before they reach a path, so `--resume ../../x` cannot read outside the folder. `list` parses every file, which is fine for hundreds of sessions. **Alternative rejected:** a separate index file, which can go out of step with the sessions.

### 3. When to save
`SessionRecorder(store, session, context, warn)`, `save()`:
- Called by `run_request` on each `ModelCallStarted`, and after the event generator closes.
- **Why those points:** at `ModelCallStarted` every tool call of earlier turns has its result. After closing, the loop's `finally` has already rolled back a cancelled request. Saving at `ToolCallFinished` would write a history with unanswered calls whenever one turn has several.
- With no messages it deletes the file if there is one, so a cancelled first request leaves nothing.
- `OSError` warns once, until a save succeeds again. `updated_at` is set on each save.

Saving is driven from the events, like the display, so the loop does not know about sessions.

### 4. Resuming
`open_session(config, cwd, resume, agent, view)` builds the recorder. With `resume` set (`""` means latest), it loads the session, calls `context.replace_messages`, and warns about a different folder or notes a different model. The session keeps its original `cwd` for the listing unless it is resumed elsewhere, where `cwd` becomes the current folder, so `--resume` finds it there next time. Provider and model are updated to the current ones on the next save.

The session is looked up before `create_agent`, so a bad id fails without a provider key or an MCP connection: `load_session(resume, cwd)` in `main.py`, importing only `nar.session`.

**`--resume` option:** a Click option with `is_flag=False, flag_value=""`. It consumes the next word, so `nar --resume "a prompt"` treats it as an id and gives "no session" with a hint to use `nar --resume <id> "prompt"`. **Alternative rejected:** guessing whether the word is an id, which would send a mistyped id to the model.

### 5. Switching
`/resume` calls `recorder.save()`, loads the target, and only then does `context.replace_messages` and `recorder.switch(session)`, so a failure keeps the current session. The approval "always" answers and the `todos` list stay as they are in the running process.

### 6. Display
`render_history(messages, console)` uses `rich.text.Text`, like the tool view, so nothing is read as markup, on standard error. Requests as `> text`, answers as text, tool calls as `-> name(...)` with arguments cut to 80 characters; tool results are left out because they can be huge and the model has them. `render_sessions` prints one line per session on standard output, as `/mcp` does, since it is the command's answer.

## Risks / Trade-offs

- **Secrets in plain text on disk** → file and folder modes on Linux and macOS, and a README note. On Windows the modes do not apply and the folder relies on the user profile's permissions.
- **Files grow forever** → a README note says they can be deleted by hand; retention is out of scope.
- **Two `nar` processes on one session** → the last save wins. Atomic writes keep each file valid.
- **A resumed conversation can mention a tool that is no longer configured** → the model is told by the registry's unknown tool error, as today.
- **Saving a big history before each model call** costs a few milliseconds compared with a model call, and compaction keeps it bounded.

## Migration Plan

Additive. Nothing to convert.

## Open Questions

None.
