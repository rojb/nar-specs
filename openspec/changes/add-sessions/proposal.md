# Proposal

- **Owner:** nathaliascode
- **Linear issue:** RRG-76
- **Video timestamps covered:** 07:32:00 (Session), 18:16:12 (Sessions vs Checkpoints), 18:18:27 (Session Management)
- **Depends on:** RRG-63 `add-agent-loop` (merged; not archived yet, see Impact)
- **Unblocks:** RRG-77 `add-checkpoints`
- **Course requirement covered:** sessions with history

## Why

Everything the agent knows about a conversation lives in memory. Closing the terminal, or a crash, loses it, and the user has to explain the work again. The course asks for sessions with history: the conversation has to be saved and the user has to be able to continue it later.

## What Changes

- **Sessions saved automatically:** each conversation (interactive or single mode) gets an id and is saved to `~/.nar/sessions/<id>.json` (`NAR_HOME` respected) with its creation and update dates, working folder, provider, model and full message history. It is saved before every model call and when a request ends, so a closed terminal loses at most the turn in progress, and the saved history is always valid for the providers.
- **`nar --resume`** continues the latest session of the current folder; **`nar --resume <id>`** continues a specific one (a unique beginning of the id is enough), and both can take a prompt for single mode.
- **`/sessions`** lists saved sessions and **`/resume [id]`** switches to another one during interactive mode.
- **On resume** the conversation is rebuilt and the user sees the earlier history. The system prompt is built again for the current folder, and the configured provider and model are used.
- **A save that fails never stops the work:** one warning, and the session goes on.

Out of scope: restoring the `todos` list or "always allow" answers (approvals are asked again on purpose), deleting or renaming sessions, removing old sessions automatically, two `nar` processes writing the same session safely, full-text search in sessions, checkpoints (RRG-77).

**"Done when" (RRG-76):** close the terminal, run `nar --resume` and continue the conversation where it stopped.

## Capabilities

### New Capabilities

- `sessions`: saving a session, what it holds, resuming it, listing it, its slash commands, how the history is shown, and failures.

### Modified Capabilities

None. The new option and commands are described in `sessions`, because the specs for interactive mode and slash commands still belong to unarchived changes.

## Impact

- **Code (`nar` repo):** new package `src/nar/session/` (`store.py`, `recorder.py`); `src/nar/cli/main.py` (`--resume`), `src/nar/cli/runner.py`, `src/nar/cli/repl.py`, `src/nar/cli/slash.py`; new `src/nar/ui/session_view.py`; README.
- **User filesystem:** writes `~/.nar/sessions/<id>.json`, readable only by the user on Linux and macOS. These files hold everything the conversation held, including file contents and command output, in plain text, and they are never removed by `nar`.
- **Dependencies:** none new.
- **Specs:** only a new capability, so it can be archived before or after `add-agent-loop`, `add-mcp-client` and `add-context-compaction`.
