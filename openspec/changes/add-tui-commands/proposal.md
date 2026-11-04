# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-78
- **Video timestamps covered:** 17:39:50 (Command support on the TUI)
- **Depends on:** RRG-63 `add-agent-loop` (merged in `nar`; not archived yet, see Impact)
- **Unblocks:** RRG-77 `add-checkpoints` and any later change that adds a command: they register it in the same table
- **Course requirement covered:** selectable model provider (the `/model` half of it: changing provider and model without leaving the session)

## Why

Interactive mode reads a line and sends it to the model. The only things that are not a request are
`exit` and the four commands the last changes added one at a time (`/mcp`, `/compact`, `/sessions`,
`/resume`), each registered in a table nobody can discover: there is no `help`, no completion, and no
way to see or change the provider, the model, the approval level, the configuration, the tools or the
context usage without leaving `nar`.

Two of those are not a convenience. Changing the model today means leaving the session and losing the
conversation, which is exactly what the course requirement asks to avoid. And a command table nobody
can list is a table that will drift, which matters because four changes are still going to add
commands to it.

## What Changes

- **A discoverable router:** `/help` lists every registered command with its one-line summary, and the
  interactive prompt completes `/` commands and offers their summaries as you type.
- **Eight base commands:** `/help`, `/clear`, `/exit`, `/model`, `/approval`, `/config`, `/tools`,
  `/tokens`. The four commands other changes already registered stay as they are.
- **`/model` changes provider and model in place:** the conversation, the tools and the session stay;
  only the next model call uses the new provider. `/approval` does the same with the approval level.
- **`/clear` starts a new conversation** in the same session, keeping the system prompt, and says so.
- **`/config`, `/tools` and `/tokens`** show what `--show-config` shows, the tool names with their
  summaries, and the context usage line.
- **Nothing changes when the input is not a terminal:** with a pipe or a redirected file the prompt
  falls back to the plain reader, so scripts and the test suite keep working.

Out of scope: editing configuration files from the TUI, completing file paths or `AGENTS.md` names,
history search and key bindings beyond what `prompt_toolkit` gives by default, `/model` per-request
(`--provider`/`--model` already exist), updating the provider recorded in a session file that was
already created, and a `/cost` command with prices (the provider does not return cost).

**"Done when" (RRG-78):** *"/model cambia de Anthropic a Ollama sin reiniciar `nar` y la conversación
continúa"* — quoted from the Linear issue, which is written in Spanish. In English: `/model` switches
from Anthropic to Ollama without restarting `nar`, and the conversation continues.

## Capabilities

### New Capabilities

- `tui-commands`: the command router, the completion, the eight base commands, what `/model` and
  `/approval` change in place, and the fallback when the input is not a terminal.

### Modified Capabilities

None. The command table already exists in `cli-entrypoint` terms (RRG-74 added the seam), and this
change adds a capability instead of rewriting that requirement, so it can be archived in any order.

## Impact

- **Code (`nar` repo):** new `src/nar/cli/commands.py` (the eight base commands, registered at import)
  and `src/nar/cli/prompt.py` (the reader with `prompt_toolkit` and its completer);
  `src/nar/cli/repl.py` (uses the reader, keeps the same loop); `src/nar/cli/runner.py`
  (`switch_model`, `clear_conversation` helpers the commands call); `pyproject.toml`
  (`prompt-toolkit` dependency); README.
- **Dependencies:** one new runtime dependency, `prompt-toolkit`, already in the project's stated
  stack. It is imported only by the interactive prompt, so `nar "..."` and `--help` never load it.
- **Existing tests:** the REPL tests feed standard input, so they keep using the plain reader; the
  ones that assert the prompt text keep passing.
- **Specs:** one new capability, no MODIFIED requirement, so no archive ordering constraint.
