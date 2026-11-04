# Design

## Context

`SessionStore` (RRG-76, `src/nar/session/store.py`) keeps one JSON file per session, and `SessionRecorder` keeps it in step with `ContextManager`, saving at points where every tool call has its result. `ContextManager.replace_messages` swaps the history and forgets the reported usage. `nar.cli.slash.register_command` is the seam that RRG-78 keeps and asks later changes to use.

Constraints:
- `SessionRecorder.save` runs before every model call, so anything added to the session file is rewritten on every turn.
- `SessionStore.list` and `find` read every `*.json` directly under `sessions/`.
- Commands run between requests, so a checkpoint never holds an unanswered tool call.

## Goals / Non-Goals

**Goals:**
- Going back is instant and needs no model call.
- A session without checkpoints costs nothing.
- A restore can be undone.

**Non-Goals:**
- Restoring files or undoing commands, automatic checkpoints, deleting checkpoints.

## Decisions

### 1. Layout
```
src/nar/session/checkpoints.py  NEW       Checkpoint, CheckpointStore
src/nar/session/store.py        MODIFIED  write_private, message_to_json, message_from_json made public
src/nar/session/recorder.py     MODIFIED  checkpoint(), restore(), checkpoints()
src/nar/ui/session_view.py      MODIFIED  render_checkpoints()
src/nar/cli/slash.py            MODIFIED  /checkpoint, /checkpoints, /restore
```
No new configuration keys.

### 2. A file of its own
`sessions/checkpoints/<session id>.json`:
```json
{"version": 1, "session": "<id>", "checkpoints": [
  {"id": 1, "name": "base", "created_at": "...", "messages": [...]}]}
```
Each checkpoint holds a full copy of the messages, in the same shape as the session file. **Why a separate file:** inside the session file every checkpoint would be rewritten before every model call. **Why a subfolder:** `SessionStore` reads `*.json` directly under `sessions/`, so a sibling file would show up as an invalid session; `glob` does not go into the subfolder. **Alternative rejected:** storing only the message count and truncating the session, which breaks as soon as a compaction replaces earlier messages. The same reason rules out sharing messages between checkpoints.

Writes use the session store's atomic write, with the same modes. A file that fails to parse raises `SessionFileError` naming it, and `add` reads before writing, so a corrupt file is never overwritten.

### 3. Numbers and names
The next number is the highest existing one plus 1. Names must not be only digits and must be unique, so `/restore 2` and `/restore base` can never be ambiguous. **Alternative rejected:** a generated id as in sessions, which is harder to type for something used inside one conversation.

### 4. Restore
`SessionRecorder.restore(ref)` finds the checkpoint first, then calls `context.replace_messages` and `save`. An unknown reference changes nothing. Later checkpoints are kept: this is what makes a restore undoable, and it replaces an automatic safety checkpoint, which would fill the list. Restore does not ask for confirmation for the same reason, as long as the user saved a checkpoint; messages after the last checkpoint are lost, and the README says so.

Token usage becomes an estimate after `replace_messages`, as after a compaction. The loop detector resets at the start of each request, so no state is carried over.

### 5. Commands
Registered with `register_command` in `slash.py`, next to `/sessions` and `/resume`. `/restore` takes the whole argument as one reference, so a name with spaces works. `NarError` and `OSError` become `view.error`. `/checkpoints` prints to standard output, as `/sessions` does. The restore line is a note on standard error, followed by the usage line.

## Risks / Trade-offs

- **Full copies grow the file** → the size of a checkpoint is bounded by the context window through compaction, the file is written only on `/checkpoint`, and the README says it can be deleted by hand.
- **Users may expect files to be restored** → every restore says they were not, and the README explains it.
- **Messages after the last checkpoint are lost on restore** → documented; a checkpoint first keeps them.
- **Two `nar` processes adding checkpoints to one session** → the last write wins, as with sessions.
- **RRG-78 may rewrite the command table** → these commands use only `register_command`, which that design keeps.

## Migration Plan

Additive. Existing session files do not change.

## Open Questions

None.
