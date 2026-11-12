# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-98
- **Video timestamps covered:** none
- **Depends on:** RRG-97 (`add-language-setting`, merged and archived)
- **Course requirement covered:** none

## Why

RRG-97 made the model answer in Spanish by default, but everything else `nar` prints is still English: the banner, `/help`, errors, warnings, prompts and `--help`. A Spanish-speaking team gets a Spanish answer inside an English interface. The `language` setting already says which language the user wants, so the interface should follow it.

## What Changes

- A message catalog with two languages, `es` and `en`. Every text the user sees in the terminal is looked up in the catalog by key. The active language is the `language` setting from RRG-97 (`language` in a file, or `--language`), and it defaults to `es`.
- These texts are translated: the banner, the command names' summaries in `/help`, errors, warnings, prompts (tool approval, trust for `.nar/tools/`, API key, provider menu), tool and context lines, and `--help` of the CLI.
- These are not translated: anything sent to the model (tool results and tool error messages, hook refusal text, tool descriptions, the system, subagent and compaction prompts), command names and flag names, configuration keys and their values, file paths, provider and model names, and anything written by hooks, MCP servers or providers.
- Answers at the prompts work in both languages, whatever the active language is: `y`, `yes`, `s`, `si`, `sí`, `n`, `no`, `a`, `always`, `siempre`, in any case and with or without the accent.
- `--help`, `--version` and Click's parse errors run before the configuration is loaded, so `nar` works out the language in a best-effort way first: `--language` in the arguments, else the configuration files (errors ignored), else `es`. Click's own words ("Usage:", "Options:", "Error:", and so on) are translated too.
- The English texts quoted in the existing specs stay as they are. With `language = es`, the user sees the Spanish equivalent of each one, with the same meaning and values.
- The README explains the interface language and how to add a text.
- Tests run with the language fixed to `en`. A new test checks that both languages have the same keys and the same placeholders.

Out of scope: more languages than `es` and `en`; changing the language in a running session; translating text that reaches the model.

**"Done when" (RRG-98):** without any configuration, the banner, `/help`, an error, a warning and a tool approval prompt are shown in Spanish. With `--language en`, they are shown in English. The catalog test finds no missing keys.

## Capabilities

### New Capabilities

- `ui-language`: the catalog and the active language, what is and is not translated, how the quoted texts in other specs relate to the catalog, bilingual prompt answers, output before the configuration is loaded, and catalog completeness.

### Modified Capabilities

- `approval-system`: the answers to the approval question are accepted in Spanish too. MODIFIED.
- `custom-tools`: the answers to the trust question are accepted in Spanish too. MODIFIED.
- `cli-entrypoint`: `--help` is shown in the language worked out before the configuration is loaded. MODIFIED.

## Impact

- **Code (`nar` repo):**
  - new package `src/nar/i18n/` (catalog, active language, shared prompt answers)
  - `src/nar/cli/main.py`, for the early language and the Click translation
  - the interface modules in `ui/`, `cli/`, `config/`, `session/`, `skills/`, `mcp/`, `hooks/`, `tools/`, `context/` and `providers/` that print text, listed in `design.md`
  - README and tests, including a new `tests/conftest.py`
- **Delivery:** five stacked PRs, described in `design.md`.
- **Dependencies:** none.
