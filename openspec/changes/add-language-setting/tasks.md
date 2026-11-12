# Tasks

> Review workload: about 350 lines, more than half tests. It fits in one PR, split into one commit per group.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-97 and verify the link opens `openspec/changes/add-language-setting/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-97 branch; verify `openspec validate add-language-setting --strict` passes

## [nar]

- [x] 1.1 Create Linear's RRG-97 branch from `main`, add the `Language` enum and the `language` setting to `NarConfig` (default `es`), and add the `--language` option to `cli/main.py`. Verify in `tests/test_config*.py` and `tests/test_cli.py` with `NAR_HOME` in `tmp_path`:
  - the default is `es` with source `default`
  - the user file, the project file and the flag each win over the previous one, and `--show-config` shows the right source
  - `language = "fr"` in a file exits with code 1 and one line naming `language` and listing `es, en`
  - `--language fr` exits with code 2 and lists both values
  - `/config` lists `language` with its source

  _Evidence: `6724f72`, 6 tests in `tests/test_config.py` and 7 in `tests/test_cli.py`, plus new assertions in the two existing `--show-config` and `describe` tests; `pytest` gave 1427 passed and 3 skipped (baseline 1414 passed and 3 skipped)._
- [x] 1.2 Add the language section to `build_system_prompt` and `build_subagent_prompt`, add the language to the summary instructions through `summary_system`, and pass `config.language` from `create_agent` to the prompt, the `Compactor` and the `TaskTool`. Verify in `tests/test_system_prompt.py`, `tests/test_subagent.py`, `tests/test_context_compaction.py` and the runner tests:
  - the system prompt names Spanish by default and English with `en`, and keeps the exceptions for code, paths, commands and quoted text
  - the first request of a subagent has a system prompt with the same section
  - the instructions of a summary name the language, for the main agent and for a subagent
  - `--language en` in single mode puts the English instruction in the prompt sent to the provider
  - existing tests that compare prompt text are updated to the new default on purpose

  _Evidence: `8bfb6d2`, 5 tests in `tests/test_system_prompt.py`, 3 in `tests/test_subagent.py`, 2 in `tests/test_context_compaction.py` and 3 in `tests/test_cli.py`. No existing test needed a change: none compared the prompt text exactly. `pytest` gave 1440 passed and 3 skipped._
- [x] 1.3 Document the setting in the README (a `language` row in the configuration table, `--language` in the flags and examples, and a short note that the interface stays in English until RRG-98). Verify `ruff check .`, `ruff format --check .` and `pytest` are clean

  _Evidence: `35808e3`, a `## Reply language` section, the `language` row in the table and `--language` in the usage list. `ruff check .` passed, `ruff format --check .` reported 156 files already formatted, and `pytest` gave 1440 passed and 3 skipped._

## Manual verification

- [x] 2.1 Walk through RRG-97's "done when" in a real terminal, in a folder without configuration:
  - run `nar "what does README.md say?"` (any question written in English) and record that the answer is in Spanish
  - run the same question with `--language en` and record that the answer is in English
  - run `nar --show-config` and record the `language` line with its source (`default`), then with `--language en` record the source `command line`
  - set `language = "en"` in `.nar/config.toml`, run `nar --show-config`, and record the project file as the source

  _Evidence (Windows 11, 2026-11-12, empty temp folder and `NAR_HOME`, Anthropic `claude-haiku-5-5`, single mode):_
  - _`nar "What is the capital of France? Answer in one short sentence."` answered `La capital de Francia es París.`_
  - _The same question with `--language en` answered `The capital of France is Paris.`_
  - _`nar --show-config` printed `language = es  (default)`, and with `--language en` it printed `language = en  (command line)`._
  - _With `language = "en"` in `.nar/config.toml`, it printed `language = en  (.nar/config.toml)`._
  - _`--language fr` printed `Error: Invalid value for '--language': 'fr' is not one of 'es', 'en'.`_
