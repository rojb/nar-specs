# Design

## Context

- `NarConfig` (`src/nar/config/models.py`) is a pydantic model with `extra="forbid"`. `load_config` builds the defaults from it, merges user file, project file and command-line overrides while recording the source of each key, and validates once. Enum fields (`provider`, `approval`) already produce the one-line error that names the key and lists the allowed values. `--show-config` and `/config` print every field of the dumped model with its source, so a new field appears there without extra code.
- `cli.main` declares `--provider`, `--model` and `--approval` as Click options and passes them to `load_config(cwd, overrides=...)`. `click.Choice` rejects an unknown value with exit code 2.
- `runner.create_agent` builds the system prompt once with `build_system_prompt(...)`, the `Compactor` and the `TaskTool`. The prompt is never rebuilt during a session, and `/clear`, `/model` and `/resume` keep it.
- `TaskTool.execute` builds each subagent prompt with `build_subagent_prompt(...)` and gives the subagent `Compactor(parent.provider, parent.compactor.limits)`.
- The summary instructions are the constant `SUMMARY_SYSTEM` in `context/compaction.py`, sent by `Compactor._summarize`.

## Goals / Non-Goals

**Goals:**
- One setting decides the language for the main agent, every subagent and the compaction summary.
- The instruction is in the prompts, not in the user's request, so the history stays as the user wrote it.

**Non-Goals:**
- No translation of `nar`'s own messages, warnings or `/help` (RRG-98).
- No change of language in a running session: the prompt is built once, as today.
- No check that the model obeys; it is an instruction, like the rest of the system prompt.

## Decisions

### 1. `Language` enum and `language` setting
`src/nar/config/models.py` (MODIFIED) adds `Language(StrEnum)` with `ES = "es"` and `EN = "en"`, and a `label` property returning `"Spanish"` or `"English"`. `NarConfig` gets `language: Language = Language.ES`, next to `approval`. Being an enum, it gets the existing validation, the error wording and the `--show-config` line for free. Alternative considered: a free string checked against a list, which would need its own error path.

### 2. `--language` option
`src/nar/cli/main.py` (MODIFIED) adds `--language` as a `click.Choice` over the `Language` values, and adds it to the `overrides` passed to `load_config`. Click rejects other values with the allowed ones listed and exit code 2, exactly as for `--approval`.

### 3. Prompt section
`src/nar/agent/prompt.py` (MODIFIED) adds `language_section(language) -> str`, a short `# Language` section built from `language.label`:

```
# Language
Always reply to the user in <Spanish|English>, whatever language the user writes in or the files are in. Keep code, identifiers, file names, paths, commands, tool arguments and quoted text unchanged.
```

- `build_system_prompt(..., language: Language = Language.ES)` (keyword) appends it after the environment section, before memory, project instructions and skills, so an `AGENTS.md` that asks for something else still comes later in the prompt.
- `build_subagent_prompt(..., language: Language = Language.ES)` appends the same section after the environment section.

### 4. Subagents and compaction
- `src/nar/agent/subagent.py` (MODIFIED): `TaskTool.__init__` gets `language: Language = Language.ES` and passes it to `build_subagent_prompt`. The subagent's `Compactor` takes the parent's language.
- `src/nar/context/compaction.py` (MODIFIED): `summary_system(language) -> str` returns `SUMMARY_SYSTEM` followed by "Write the summary in <language>. Keep file paths, names, commands and values exact." `Compactor.__init__` gets `language: Language = Language.ES`, stored on the instance and used in `_summarize`. `SUMMARY_SYSTEM` stays as the language-free base.
- `src/nar/cli/runner.py` (MODIFIED): `create_agent` passes `config.language` to `build_system_prompt`, `Compactor` and `TaskTool`.

### New configuration key
- `language`, in `~/.nar/config.toml` or `.nar/config.toml`: `"es"` or `"en"`, default `"es"`.

## Risks / Trade-offs

- **The default changes behavior for everyone** → without configuration, answers are now in Spanish. This is the intent of RRG-97; `language = "en"` or `--language en` restores English.
- **The model may still reply in another language** → the instruction is a request, not a filter. It sits early in the prompt and is short to keep it effective.
- **Conflict with `AGENTS.md`** → project instructions come after the language section, so they can refine it, for example "answer commit messages in English".
- **Existing tests that compare prompt text** → they now see the language section; they are updated to the new default on purpose.

## Migration Plan

None. Existing configuration files stay valid. A user who wants the old behavior of answering in the language of the question has no equivalent; `en` is the closest.
