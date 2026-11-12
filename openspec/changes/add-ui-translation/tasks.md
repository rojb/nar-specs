# Tasks

> Review workload: more than 400 changed lines in total, so it ships as five stacked PRs, one per group 1 to 5 below. Slice N branches from slice N-1. Branch names are `rubenjustiniano96/rrg-98-ui-translation-<n>-<short>`. Every slice verifies `ruff check .`, `ruff format --check .` and `pytest`.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-98 and verify the link opens `openspec/changes/add-ui-translation/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-98 branch; verify `openspec validate add-ui-translation --strict` passes

## [nar]

### 1. Catalog core

Branch: `rubenjustiniano96/rrg-98-ui-translation-1-core`, from `main`.

- [x] 1.1 Create `src/nar/i18n/` with `t(key, **values)`, `set_language` and `current_language()`, and the two message modules `messages_es.py` and `messages_en.py` with the first keys. Verify in `tests/test_i18n.py`: (c7e8f37; `tests/test_i18n.py` 7 tests (format, default `es`, unknown key names the key, set_language))
  - `t` formats the placeholders and uses the active language
  - the default language is `es` before `set_language` is called
  - an unknown key raises an error that names the key
- [x] 1.2 Add the parity test: both catalogs have exactly the same keys, and each key has the same placeholders in both. Verify it fails on a missing key and on a different placeholder (two small fixtures inside the test) (3fd1206; parity test plus two fixtures (missing key, different placeholder), 3 tests)
- [x] 1.3 Add `tests/conftest.py` with an autouse fixture that sets the language to `en` before each test and restores the default after it. Verify the existing suite passes with no other test change (929ab7d; suite 1451 passed, 3 skipped with no other test change (baseline 1440 + 11 new))
- [x] 1.4 Add `src/nar/i18n/answers.py` (`parse_answer`: yes, no, always; case and accent insensitive; `y`, `yes`, `s`, `si`, `sí`, `n`, `no`, `a`, `always`, `siempre`). Verify in `tests/test_i18n.py` with a table of accepted and rejected answers, including `SÍ`, ` Siempre ` and `quizás` (42c8c2a; 28 answer cases (`SÍ`, ` Siempre `, `quizás` and more))
- [x] 1.5 Call `set_language(config.language)` in `cli()` right after `load_config`. Verify with a test that runs `cli` with `--language en` and with the default, and checks `current_language()` (c60decb; 2 tests in `tests/test_cli.py` (`--language`, file and default))
- [x] 1.6 Document in the README the interface language and the steps to add a text (add the key to both message modules, use `t(...)`, run the parity test). Verify `ruff check .`, `ruff format --check .` and `pytest` are clean (fb61d59; README section "Interface language"; ruff check, ruff format --check clean, pytest 1481 passed, 3 skipped)

### 2. Prompts and views

Branch: `rubenjustiniano96/rrg-98-ui-translation-2-prompts`, from slice 1.

- [x] 2.1 Translate `ui/approval_prompt.py`, `ui/trust_prompt.py` and `ui/key_prompt.py`, and make them use `parse_answer`. Verify in the prompt tests, with `es` set: (604b42d; `tests/test_prompts_i18n.py` 47 cases: Spanish and English questions, `YES`/`sí`/`SI`/`siempre` in both languages, `quizás` asks again, rejected and unavailable results stay English; `tn` added for plural keys)
  - the three questions read as in the `ui-language` spec (`¿Permitir write_file? [s]í / [n]o / [a] siempre write_file en esta sesión: `)
  - `YES`, `sí`, `SI` and `siempre` are accepted in both languages, `quizás` asks again
  - the rejected and unavailable results sent to the model stay English in `es`
- [x] 2.2 Translate `ui/provider_menu.py` and the labels in `cli/fallback.py`. Verify with `es` set that the menu text is Spanish and the provider names and the configuration values are unchanged (03fd732; menu, status labels and `Elige 1-3:` in Spanish, names, models and values unchanged)
- [x] 2.3 Translate `ui/tool_view.py`, `ui/agent_view.py`, `ui/session_view.py` and `ui/mcp_view.py`, with the `aviso:` prefix and separate keys for plurals. Verify with `es` set: a tool result summary and an agent warning are Spanish, the output of the tool that goes to the model is English, and `1` and `2` invalid session files give the two sentences of the spec (8185236; `tests/test_views_i18n.py` 25 tests: tool summary, `aviso:` warnings, loop and context lines, `1`/`2` invalid session files give `1 archivo de sesión no es válido y se omitió` and `2 archivos de sesión no son válidos y se omitieron`, model-bound output unchanged)
- [x] 2.4 Document the prompt answers in the README. Verify `ruff check .`, `ruff format --check .` and `pytest` are clean (2c4982d; README table of questions and accepted answers; `ruff check .`, `ruff format --check .` clean, `pytest` 1558 passed, 3 skipped)

### 3. Slash commands and REPL/runner

Branch: `rubenjustiniano96/rrg-98-ui-translation-3-commands`, from slice 2.

- [x] 3.1 Make `register_command` take a catalog key and resolve it when commands are listed (`cli/slash.py`, `cli/commands.py`). Verify with `es` and then `en` in the same process that `/help` lists the summaries in each language and the command names do not change (de3b9e0; `tests/test_commands_i18n.py` 3 tests: `/help` and completion list the summaries in `es` then `en` in one process, command names unchanged; `register_command` takes a catalog key, 18 summary keys)
- [x] 3.2 Translate the output of the commands in `cli/commands.py` and `cli/slash.py`. Verify with `es` set that `/config`, `/tools`, `/tokens`, `/model`, `/sessions` and `/skills` print Spanish sentences with the same keys, values and sources (eff6a3b; 38 `cmd.*` keys; Spanish tests in `tests/test_cli.py` and `tests/test_slash.py`: `/config`, `/tools`, `/tokens`, `/model`, `/sessions`, `/skills` and the errors print Spanish with the same keys, values and sources; `/tools` descriptions stay English; singular fixes in `en`: `1 message`, `1 model`)
- [x] 3.3 Translate the banner and notices in `cli/repl.py`, the messages in `cli/runner.py` and the prompts in `cli/api_key.py`. Verify with no configuration that the banner is `nar · <provider> · <model>. Escribe 'exit' para salir.` and with `--language en` it is the English one (168e95b; banner `nar · fake · fake-model. Escribe 'exit' para salir.` with no config, English with `--language en`; waiting text, runner and key notes translated; the `summary` of the approval rejected and unavailable results translated, their `output` stays English)
- [x] 3.4 Document the translated commands in the README. Verify `ruff check .`, `ruff format --check .` and `pytest` are clean (1572c46; README "Commands"; `ruff check .`, `ruff format --check .` clean, `pytest` 1580 passed, 3 skipped)

### 4. Module errors and warnings

Branch: `rubenjustiniano96/rrg-98-ui-translation-4-errors`, from slice 3.

- [x] 4.1 Add `SessionNotFound` in `session/store.py` and use it in `main._find_session` instead of matching the message prefix. Verify with `es` set that `nar --resume zzz` still adds the hint about giving a prompt, and that an unrelated session error does not (78f2d0f; `SessionNotFound` raised for no session and invalid id, `_find_session` catches it; `tests/test_module_errors_i18n.py`: `nar --resume zzz` in `es` adds the `solicitud` hint, a corrupt file does not; the `--resume` notes in `main.py` translated too, `1 message` singular fixed in `en`)
- [x] 4.2 Translate the errors of `config/loader.py` (and the source words used by `describe`), `session/*`, `skills/loader.py` and `context/instructions.py`. Verify with `es` set: `language = "fr"` in a file gives one Spanish line that names `language` and lists `es, en`, with exit code 1, and the key names and values stay as they are (6858c4a; `language = "fr"` in a file gives one `error: valor no válido "fr" para el ajuste 'language' ...; valores permitidos: es, en` line with exit code 1; source words `predeterminado` / `línea de comandos` in `--show-config` and `/config`; session, checkpoint, skill and `AGENTS.md` errors in Spanish; `SkillError.model_message` keeps the `skill` tool result English; `en` skill warning now starts with `warning:` as the spec says)
- [x] 4.3 Translate the warnings and notes of `mcp/manager.py`, `hooks/runner.py`, `tools/custom.py` and the compaction errors, and the `RetryNotice` text in `providers/retry.py`. Verify with `es` set that each starts with `aviso:` where it is a warning, and that the text of a hook refusal reaching the model stays English (2115f75; `aviso:` on MCP, hook, custom tool and recorder warnings, `RetryNotice` reworded (`reintentando openrouter en <n>s (error del servidor 503, intento 2/4)`), compaction errors translated; the `before_tool` refusal text stays English; singular fixed in `en`: `1 tool file`)
- [x] 4.4 Translate the preview titles and summaries in `tools/edit_file.py`, `tools/write_file.py` and `tools/shell.py` only. Verify with `es` set that the preview title is Spanish and the result sent to the model is the same as with `en` (8ee2f5f; `Editar a.py (+1 -1)`, `Crear a.txt (1 línea)`, `Ejecutar en bash (límite 7 s)` and the summaries translated, `ToolPreview.error` and all model output unchanged; singular fixed in `en`: `1 line`)
- [x] 4.5 Document the rule "what is translated and what is not" in the README. Verify `ruff check .`, `ruff format --check .` and `pytest` are clean (878b544; README "What is translated and what is not"; `ruff check .`, `ruff format --check .` clean, `pytest` 1608 passed, 3 skipped)

### 5. Click and pre-config output

Branch: `rubenjustiniano96/rrg-98-ui-translation-5-click`, from slice 4.

- [x] 5.1 Pre-resolve the language in `main()` (`--language` in the arguments, else the config files ignoring errors, else `es`) and build the Click command after it, with the option help texts from the catalog. Verify in the CLI tests: `--help` is Spanish by default, English with `--language en` and with `language = "en"` in a file, and Spanish when the config file is broken (b24b415; `preresolve_language` (`--language`/`--language=`, else project then user file ignoring every error, else `es`; an unknown value counts as not given) runs in `main()` before Click; the command is built by `build_command()` after it, option help from 8 `cli.*` keys; `tests/test_click_i18n.py`: `--help` Spanish by default, English with `--language en` and with `language = "en"` in a file, Spanish with a broken config file, option names and choices identical in both)
- [x] 5.2 Install the Click translation (`nar/i18n/click_words.py`) for "Usage:", "Options:", "Show this message and exit.", "Error:", "Invalid value for ...", "No such option" and "Try '...' for help.". Verify `nar --foo` and `nar --language fr "hi"` print Spanish errors with exit code 2, and add a test that pins the Click message ids used (072965f; `nar/i18n/click_words.py` replaces `gettext.translation` while the command runs (verified with Click 8.5.0: every Click module binds `gettext` at import, but `gettext.gettext`/`ngettext` look `translation` up at call time) and answers 19 Click ids from the catalog, unknown ids fall back to English; real runs: `nar --approval bogus` and `nar --nope` print `Uso:`, `Prueba 'nar --help' para ver la ayuda.` and a Spanish `Error:` with exit code 2, `nar --language fr "hi"` lists `'es', 'en'`; pins: English catalog texts equal Click's ids and a test that Click still asks for every id in the table)
- [x] 5.3 Translate the `main()` error boundary (`cancelled`, the unexpected failure line and the `NAR_DEBUG` hint) and show a config error in the pre-resolved language. Verify `language = "en"` in the user file plus an invalid `approval` in the project file gives the English error with exit code 1 (c50e524; `main()` boundary through the catalog (`error: fallo inesperado (...)`, `pista: define NAR_DEBUG=1 ...`, `cancelado`); `language = "en"` in the user file plus `approval = "bogus"` in the project file gives the English `error: invalid value ...` with exit code 1, with no language anywhere it is Spanish; also the carry-overs: `ProviderError` words that nar writes (retry/adapters/factory) translated with an English `model_message` for subagents, and the approval `(vista previa no disponible ...)` note)
- [x] 5.4 Document `--help` and the early language in the README. Verify `ruff check .`, `ruff format --check .` and `pytest` are clean (a4828a5; README "Interface language" > "Before the configuration is loaded" with the order, real `--help`/error examples and the Click mechanism; `ruff check .`, `ruff format --check .` clean, `pytest` 1669 passed, 3 skipped)

## Manual verification

- [x] 6.1 Walk through RRG-98's "done when" in a real terminal, in a folder without configuration, and record what is observed:
  - run `nar` and record that the banner is in Spanish; run `/help` and record Spanish summaries
  - cause an error (for example `nar --resume zzz`) and record the Spanish message with the `error:` prefix
  - cause a warning (for example a broken skill file or an invalid session file) and record the `aviso:` prefix
  - ask the model for something that needs approval at the `ask` level and record the Spanish approval prompt; answer `si`, then answer `yes` to another one
  - repeat the banner, `/help`, the error, the warning and the approval prompt with `--language en` and record that all are English
  - run `nar --help` and record Spanish help; run `nar --language en --help` and record English help; run `nar --foo` and record the Spanish Click error with exit code 2
  - run the catalog test (`pytest tests/test_i18n.py`) and record that it finds no missing keys

  _Evidence (Windows 11, 2026-11-12, real console through the e2e harness, temp folder and `NAR_HOME`, Anthropic `claude-haiku-5-5`, slice-5 branch):_
  - _Banner: `nar · anthropic · claude-haiku-5-5. Escribe 'exit' para salir.` `/help` showed lines such as `/clear  inicia una conversación nueva, conservando proveedor y aprobación`._
  - _Error: `nar --resume zzz` printed `error: no hay ninguna sesión 'zzz'; para dar también una solicitud, usa nar --resume <id> "solicitud"` (exit 1). `/nope` printed `error: /nope no es un comando; disponibles: ...`._
  - _Warning: a broken skill printed `aviso: se omite un skill: ...SKILL.md: falta el encabezado (el archivo debe empezar con ---)`._
  - _Approval: `¿Permitir write_file? [s]í / [n]o / [a] siempre write_file en esta sesión:`. `si` gave `✓ creado uno.txt (1 línea)`, and `yes` gave `✓ creado dos.txt (1 línea)`. `n` gave `Motivo (opcional, Enter para omitir):`, then `✗ rechazado por el usuario`._
  - _With `--language en`, the output was `Type 'exit' to leave.`, `/clear  start a new conversation, keeping provider and approval`, `error: /nope is not a command; ...`, `warning: skipping skill: ...`, `Allow write_file? [y]es / [n]o / [a]lways allow write_file this session:` and `✗ rejected by user`._
  - _`nar --help` printed `Uso: nar [OPTIONS] [PROMPT]` and `nar: un agente de programación para la terminal.` With `--language en --help` the output was `Usage: nar [OPTIONS] [PROMPT]` and `nar: a CLI coding agent.`_
  - _`nar --foo` printed `Error: No existe la opción '--foo'.` (exit 2). `--approval bogus` printed `Error: Valor no válido para '--approval': 'bogus' no es una de las opciones: 'ask', 'auto-edit', 'yolo'.` (exit 2)._
  - _`pytest tests/test_i18n.py`: 43 passed._
  - _Still English, as designed: `[OPTIONS]`, `[PROMPT]`, `TEXT`, and provider, MCP and hook text._
