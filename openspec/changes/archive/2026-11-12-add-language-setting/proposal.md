# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-97
- **Video timestamps covered:** none
- **Depends on:** none
- **Course requirement covered:** none

## Why

`nar` answers in whatever language the model picks, which is usually the language of the question. A Spanish-speaking team wants answers in Spanish by default, even when a question or a file is in English, and wants a way to switch to English. Today the only way is to write "always answer in Spanish" in `AGENTS.md` of every project.

## What Changes

- A new setting `language`, with the values `es` and `en` and the default `es`. It is set in `~/.nar/config.toml` or `.nar/config.toml`, or for one run with `--language es|en`. The usual precedence applies: default, user file, project file, command line.
- The system prompt tells the model to always reply to the user in the configured language, and to keep code, identifiers, file names, paths, commands, tool arguments and quoted text unchanged.
- The prompt of every subagent carries the same instruction, so a subagent report reaches the main agent in the configured language.
- The instructions for the compaction summary carry the same instruction, so the summary is written in the configured language.
- `--show-config` and `/config` show `language` with its source, like every other setting.
- An unknown value in a file is a configuration error (exit 1). An unknown value after `--language` is rejected like an unknown `--approval` (exit 2).
- The language does not change during a session.

Out of scope: translating the interface of `nar` (messages, warnings, `/help`), which is RRG-98; other languages; a command to change the language in a running session.

**"Done when" (RRG-97):** without any configuration, a question written in English is answered in Spanish. With `--language en`, it is answered in English. `--show-config` shows `language` with its source.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `configuration`: the new `language` setting, its validation and the `--language` override. ADDED, and the list of initial settings is MODIFIED.
- `cli-entrypoint`: the `--language` option. ADDED.
- `agent-context`: the system prompt tells the model which language to reply in. MODIFIED.
- `subagents`: the prompt of a subagent tells it which language to reply in. ADDED.
- `context-compaction`: the summary is written in the configured language. ADDED.

## Impact

- **Code (`nar` repo):**
  - `src/nar/config/models.py`, for the `language` setting
  - `src/nar/cli/main.py`, for `--language`
  - `src/nar/agent/prompt.py`, for the language section in the system prompt and the subagent prompt
  - `src/nar/agent/subagent.py`, `src/nar/context/compaction.py` and `src/nar/cli/runner.py`, to pass the language on
  - README and tests
- **Dependencies:** none.
