# Design

## Context

`/model` with no argument is handled by `_model` in `src/nar/cli/commands.py`. It prints `agent.provider.name` and `agent.provider.model` and nothing else. `/config` already knows where each setting came from: `LoadedConfig.provenance` maps a key such as `provider` to `default`, a config file path, or `command line`, and `LoadedConfig.describe` turns that into the text the user sees. The slash context already holds the `LoadedConfig` as `context.loaded`.

After `/model <provider>`, `switch_model` in `src/nar/cli/runner.py` replaces `agent.provider`. The loaded config is not updated, so the provenance still describes the startup state. The agent does not record that a switch happened.

## Goals / Non-Goals

**Goals:**
- `/model` shows where the provider and the model in use came from.
- The wording is the one `/config` uses, plus one new source for a switch.

**Non-Goals:**
- Changing `/config` output or the config files.
- Tracking more than one switch. Only the last state matters.

## Decisions

### 1. Layout
```
src/nar/agent/loop.py      MODIFIED  Agent gets `switched: bool = False`
src/nar/cli/runner.py      MODIFIED  switch_model sets `agent.switched = True` after the provider is assigned
src/nar/cli/commands.py    MODIFIED  _model prints the provider and the model, each with its source
tests/test_slash.py        MODIFIED  /model output with each source
tests/test_cli.py          MODIFIED  `--provider` then /model, and /model after a switch
```

### 2. Reuse the `/config` source text
`_model` reads `context.loaded.provenance.get("provider", DEFAULT_SOURCE)` and the same for `model`, then formats the path with the helper `describe` already uses (`_display_source`, made importable under a public name if it stays private). A key missing from the provenance is `default`, because `model` is resolved later and has no entry until a file or flag sets it. The user-facing text is therefore `default`, a config file path (relative to the working directory when it is inside it), or `command line`.

The task text and the issue speak of "user config file", "project config file" and "CLI flag". The code says `default`, the file path and `command line`. This change follows the code, so `/model` and `/config` never disagree.

### 3. The agent records the switch
`switch_model` sets `agent.switched = True` as its last step, after the new provider is assigned. A failed or cancelled switch raises before that line, so the flag and the sources stay as they were. When `agent.switched` is true, `_model` prints `this session (/model)` for both lines and ignores the provenance.

Output (two lines on standard error, like the rest of `/model`):
```
provider anthropic  (command line)
model claude-haiku-5-5  (default)
```
and after `/model openrouter`:
```
provider openrouter  (this session (/model))
model <default model>  (this session (/model))
```

When `context.loaded` is `None` (a session built without a config), `_model` prints the old text without sources and does not fail.

### 4. No new config keys
There are no new keys in `~/.nar/config.toml` or `.nar/config.toml`.

## Risks / Trade-offs

- **The flag is a single bool, not a history** → enough for the spec; a second switch gives the same source.
- **Tests that match the old one-line output break** → they are updated in the same commit.

## Migration Plan

None. Nothing stored changes.

## Open Questions

None.
