# Design

## Context

- RRG-97 added `NarConfig.language` (`Language` enum, `es` or `en`, default `es`) and `--language`. Today it only reaches the model prompts. See `proposal.md` for the motivation.
- Every user-facing text is a string literal in the module that prints it. Some are built from parts or captured at import time: the `/help` summaries are passed to `register_command(...)` when `cli/slash.py` and `cli/commands.py` are imported, and the Click options in `cli/main.py` get their `help=` when the module is imported.
- `cli.main` is a Click command that calls `load_config` inside `cli()`. `--help`, `--version` and parse errors finish before `cli()` runs, so they cannot read `NarConfig`.
- Errors are `NarError` subclasses (`ConfigError`, `SessionFileError`, `SkillError`, and so on) carrying a `message`. `main.main` prints `error: <message>`. `main._find_session` decides whether to add a hint by checking whether the message starts with `no session` or `invalid session id`, which would break as soon as the message is translated.
- Some texts reach both the user and the model (for example the output of a tool is shown and also returned to the model). Those must stay English.
- Prompts: `ui/approval_prompt.py`, `ui/trust_prompt.py` and `ui/key_prompt.py` each keep their own set of accepted answers (`{"y", "yes"}` and so on).

## Goals / Non-Goals

**Goals:**
- One place where texts live, with a test that makes a missing translation impossible to miss.
- No text sent to the model changes, so model behavior and the existing tests of tools, hooks and prompts stay valid.
- A change that is reviewable in small, ordered PRs.

**Non-Goals:**
- No third language and no catalog files loaded from disk: the catalog is Python data inside the package.
- No change of language in a running session: the language is set once.
- No new configuration key: `language` from RRG-97 is reused.
- No translation of docstrings, log output for developers, or comments.

## Decisions

### 1. Catalog package `nar/i18n/` (new)
- `nar/i18n/__init__.py`: `t(key, **values)`, `set_language(language)`, `current_language()`.
- `nar/i18n/messages_es.py` and `nar/i18n/messages_en.py`: one `MESSAGES: dict[str, str]` each, with identical keys and identical placeholder names.
- `t` finds the text of the active language and formats it with `str.format`. An unknown key raises an internal error that names the key; it never returns an empty string.
- The active language is a module-level value, set once in `cli()` right after `load_config` returns, and `es` before that. A module-level value is chosen over passing the language through every constructor because texts are printed from many layers (tools, hooks, MCP, providers) that have no access to the config. It is read at call time, never captured at import time.
- Keys are whole sentences. Plurals are separate keys (`session.invalid.one`, `session.invalid.other`) chosen by the caller, so each language writes its own grammar.
- Alternatives considered: `gettext` `.po`/`.mo` files (needs a compile step and extraction tooling for 2 languages and about two hundred texts); passing a `Translator` object everywhere (a large signature change in every module).

### 2. What stays English
- Text that goes to the model stays a plain string in the module that builds it: `ToolResult.output`, tool error messages, the refusal text of hooks, the `_rejected` and `_unavailable` approval results, tool descriptions, and the system, subagent and compaction prompts.
- Only fields that are for the user alone are translated: `ToolResult.summary` and the `ToolPreview` title/detail labels.
- An exception that is only shown to the user (`ConfigError`, `SessionFileError`, `SkillError`, errors raised by the CLI and commands, `RetryNotice`) is built with `t(...)`. An exception whose message also reaches the model stays English.
- Alternative considered: translate at the display layer by matching English text. Rejected: it couples the UI to wording.

### 3. `_find_session` stops matching text
`SessionFileError` gets a distinct subclass, `SessionNotFound` (modified `session/store.py`), raised for "no session" and "invalid session id". `main._find_session` catches that class for the `--resume` hint instead of checking `message.startswith(...)`.

### 4. Shared prompt answers `nar/i18n/answers.py` (new)
- `parse_answer(text) -> Answer | None` where `Answer` is `YES`, `NO` or `ALWAYS`. It trims, lowercases and removes accents (`unicodedata` NFD, dropping combining marks), then checks: yes = `y`, `yes`, `s`, `si`; no = `n`, `no`; always = `a`, `always`, `siempre`.
- `ui/approval_prompt.py`, `ui/trust_prompt.py` and `ui/key_prompt.py` use it, each accepting only the answers its question offers (trust and key accept no `ALWAYS`). The accepted set does not depend on the active language; only the prompt wording does.
- Spanish prompt text uses `[s]í / [n]o` and `[a] siempre`.

### 5. Slash command summaries
`register_command(name, handler, summary_key)` takes a catalog key instead of the English summary. `slash.commands()` resolves the key with `t(...)` when it lists commands, so the result follows the language set in `cli()`. This removes the import-time capture.

### 6. Output before the configuration is loaded
- `main()` pre-resolves the language best-effort before Click runs: scan `argv` for `--language <v>` or `--language=<v>`; else read the user and project config files, ignoring every error and reading only `language`; else `es`. An unknown value counts as unresolved and falls through to `es`. The result is passed to `set_language`; `cli()` sets it again from the validated config.
- The Click command is built by a function after that pre-resolution, so the option help texts come from the catalog. The `--language` option help names the allowed values.
- Click's own words come from its `gettext` calls. A `NullTranslations` subclass backed by the catalog is installed so that Click's message ids map to catalog texts (for example `Usage:`, `Options:`, `Show this message and exit.`, `Error`, `Invalid value for {param_hint}: {message}`, `No such option: {name}`, `Try '{command} {option}' for help.`). The installation is done by replacing `gettext.translation` for the Click domain, which every Click `gettext` call goes through, and it is removed or reset by tests. Ids that are not in the catalog fall back to Click's English.
- `main()` error boundary: `error: <message>` (`error:` stays in both languages), `cancelled` and `hint: set NAR_DEBUG=1 ...` come from the catalog. A `ConfigError` that makes the language unknowable is printed in the pre-resolved language.

### 7. Prefixes
`error:` is the same in both languages. `warning:` is `aviso:` in Spanish. The warning texts are catalog keys that include their own prefix through a shared `warning.prefix` key, so the prefix is not repeated in each sentence.

### 8. Tests
- `tests/conftest.py` (new): an autouse fixture sets the language to `en` before each test and restores `es` after it, so the existing tests keep passing without edits.
- `tests/test_i18n.py` (new): the parity test (same keys, same placeholders per key, using `string.Formatter().parse`), unknown key, formatting, and the answers table.
- Each slice adds Spanish tests for its area with `set_language("es")`.
- A test pins the Click message ids the catalog relies on, so a Click upgrade that renames one fails loudly.

## Modules by slice

Delivery plan: five stacked PRs, in this order. Slice N branches from slice N-1.

1. **Catalog core**
   - new: `nar/i18n/__init__.py`, `nar/i18n/messages_es.py`, `nar/i18n/messages_en.py`, `nar/i18n/answers.py`, `tests/conftest.py`, `tests/test_i18n.py`
   - modified: `cli/main.py` (`set_language` right after `load_config`), `README.md` (language and how to add a text)
   - no visible change yet, except where a text is trivially wired.
2. **Prompts and views**
   - modified: `ui/approval_prompt.py`, `ui/trust_prompt.py`, `ui/key_prompt.py`, `ui/provider_menu.py`, `cli/fallback.py` (labels), `ui/tool_view.py`, `ui/agent_view.py`, `ui/session_view.py`, `ui/mcp_view.py`
3. **Slash commands, REPL and runner**
   - modified: `cli/slash.py`, `cli/commands.py` (summaries become catalog keys resolved when listed), `cli/repl.py` (banner and notices), `cli/runner.py`, `cli/api_key.py`
4. **Module errors and warnings**
   - modified: `config/loader.py` (and the source words in `describe`), `session/store.py`, `session/recorder.py`, `session/checkpoints.py`, `skills/loader.py`, `mcp/manager.py`, `hooks/runner.py` (warn and note lines, not the refusal text), `tools/custom.py`, `context/instructions.py`, `context/compaction.py` (user-facing errors), `providers/retry.py` (`RetryNotice`), `tools/edit_file.py`, `tools/write_file.py` and `tools/shell.py` (preview titles and summaries only)
   - fix: `_find_session` through `SessionNotFound`.
5. **Click and pre-config output**
   - modified: `cli/main.py` (early language, command built after it, `main()` error boundary), plus the Click translation installer in `nar/i18n/` (new file `nar/i18n/click_words.py`)

Every slice lands its own tests and README text. The estimated size is above 400 changed lines in total, which is why it is split. Slices 2 to 4 are mostly mechanical replacement of string literals.

### New configuration keys
None. The change reuses `language`.

## Risks / Trade-offs

- **A string shared with the model gets translated by mistake** → the rule in Decision 2 is applied per slice with tests that assert the model-bound result stays English in Spanish mode (a tool error, a rejection, a hook refusal).
- **Import-time captured strings keep the wrong language** → texts are resolved when shown, never stored as module constants. Slash summaries use keys (Decision 5). A test imports the modules and then sets the language.
- **Test churn** → the autouse `en` fixture keeps existing assertions valid; only tests that exercise `es` are new.
- **Click's gettext ids change between versions** → a test pins the ids the catalog uses, and unknown ids fall back to English instead of failing.
- **Pre-resolution disagrees with the loaded config** → the pre-resolved value is only used until `load_config` succeeds; then the validated value is set. The only visible effect is for output produced before that point.
- **Mixed language in one screen** → text from hooks, MCP and providers is left as written, so a Spanish screen can show an English line from them. This is intended.
- **Long Spanish texts** → prompts are short and plain; the catalog test does not check length.

## Migration Plan

None. No file format changes. Existing configuration stays valid. Users who want the old English interface set `language = "en"` or use `--language en`. Each PR is reverted independently, from the top of the stack down.
