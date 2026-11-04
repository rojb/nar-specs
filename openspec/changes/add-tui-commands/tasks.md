# Tasks

> Review workload: about 400 lines of code and docs plus 450 lines of tests, over the ~400-line guide
> (a planning heuristic, not a cap; tests are never dropped to fit). Split into two PRs: PR A = groups
> 2 and 3 (the prompt reader and the eight commands, not wired), PR B = group 4 to 6 (the REPL, the
> docs and the manual check), each retargeted to `main` before its parent merges. Branch name and
> "done when" are the ones the Linear issue shows, read on 2026-11-04.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-78 (move the issue from Backlog to In Progress and add the `nar-specs` change path as a link on the issue); verify the link opens `openspec/changes/add-tui-commands/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-78 branch with proposal, specs, design and tasks; verify `openspec validate add-tui-commands --strict` passes

## [nar]

## 1. Branch

- [ ] 1.1 In `nar/`, update `main` and create `rubenjustiniano96/rrg-78-add-tui-commands-comandos-slash-en-la-tui-help-model-clear` (the `gitBranchName` of RRG-78); verify a clean tree, that `pip install -e ".[dev]"` works, and that `pytest -q` and `ruff check .` pass before any change _Evidence: <baseline>_

## 2. Prompt and completion (PR A)

- [ ] 2.1 Add `prompt-toolkit` to `dependencies` in `pyproject.toml`, run `pip install -e ".[dev]"`, and create `src/nar/cli/prompt.py` with `read_line()`, `SlashCompleter` and the terminal branch of design.md §2; expose the command table from `src/nar/cli/slash.py` as `commands()` returning the names with their summaries in alphabetical order; new `tests/test_cli_prompt.py`; verify `pytest tests/test_cli_prompt.py` passes for: `read_line` falls back to `input()` when `sys.stdin.isatty()` is false (monkeypatched) and loads no `prompt_toolkit` module (`sys.modules`), the completer offers `/compact` and `/config` for `/co` with their summaries, offers every registered name for `/`, offers nothing for `hello` or `/etc/hosts`, leaves a second word alone, and `python -c "import nar.cli.main, sys; assert 'prompt_toolkit' not in sys.modules"` exits 0
## 3. The eight commands (PR A)

- [ ] 3.1 Create `src/nar/cli/commands.py` with the eight handlers and their `register_command` calls (design.md §3), add `switch_model` and `clear_conversation` to `src/nar/cli/runner.py`, and extend `tests/test_slash.py`; verify `pytest tests/test_slash.py` passes for: `/help` lists all twelve commands alphabetically with their summaries on standard error and nothing on standard output, `/help extra` runs anyway, `/model` alone shows provider, model and source without changing anything, `/model ollama` switches and the next request reaches the new provider, `/model ollama/qwen2.5-coder` and `/model ollama qwen2.5-coder` switch to both, `/model nope` errors listing the providers and changes nothing, `/model anthropic` without a key errors in one line and keeps the previous provider, `/approval` alone shows the level, `/approval yolo` changes it so the next shell call does not ask, `/approval nope` lists the three levels, `/clear` empties the conversation, says so, rewrites the session file with no messages and keeps provider and approval, `/config` shows the masked values and no secret, `/tools` lists built-in and `mcp__` tools with their descriptions, `/tokens` prints the usage line, `/exit` ends the session with code 0, and every command writes only to standard error
- [ ] 3.2 Wire the reader into `src/nar/cli/repl.py` and keep the loop unchanged; extend `tests/test_cli.py`; verify `pytest tests/test_cli.py` passes with every existing REPL test unchanged (they feed standard input and take the fallback) and for: `/help` then `exit` in a piped run prints the help and never calls the provider, `/exit` exits with code 0 like `exit`, `/clear` then a request sends only the new request to the provider, `/model ollama` then a request reaches the second provider with the earlier conversation intact, and an unknown command does not end the session

## 4. Docs and checks (PR B)

- [ ] 4.1 Update `nar/README.md` with a `Commands` table (all twelve, with what each does), how `/model` and `/approval` change the session in place, that completion needs a terminal, and that the session file keeps the provider it was created with; verify every command in the section runs as written in a terminal and in a pipe
- [ ] 4.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass with the only skip being the one recorded in 1.1 _Evidence: <commit>_

## 5. Delivery

- [ ] 5.1 For PR A and PR B: run `ruff check .`, `ruff format --check .` and `pytest`; commit with Conventional Commit messages (for example `feat(cli): complete slash commands at the prompt (RRG-78)`, `feat(cli): add the base session commands (RRG-78)`), push and open the PRs referencing RRG-78, retargeting PR B to `main` before merging PR A
- [ ] 5.2 Merge both PRs and confirm `main` contains the branch with `pytest` and `ruff check .` clean _Evidence: <merge commits>_

## 6. Manual verification (RRG-78 "done when")

- [ ] 6.1 Walk through RRG-78's "done when" ("/model cambia de Anthropic a Ollama sin reiniciar `nar` y la conversación continúa": `/model` switches from Anthropic to Ollama without restarting `nar`, and the conversation continues) in a real terminal at the `nar/` repo root with `NAR_HOME` pointing at a temp folder, and record what was observed for each step:
  - Start with `ANTHROPIC_API_KEY` set and Ollama running, ask a question that has an answer the next one needs (for example a code name), and record the answer
  - Completion: type `/` and record the list offered, then type `/co` and record the two commands with their summaries
  - Switch: type `/model ollama`, record the line, then ask for the code name and record that Ollama answered and remembered it
  - Back: `/model anthropic` and record the same conversation continuing with the other provider
  - Errors: `/model nope`, `/model anthropic` with the key unset, `/approval nope`, and record the messages and that the session survived each one
  - The rest: `/help`, `/config` (a masked secret present), `/tools` (including `mcp__` names when a server is configured), `/tokens`, `/clear` (and that the next answer does not remember), `/exit`
  - Pipe: `printf '/help\nexit\n' | nar` and record that it works without completion
  - Recording: date, OS and terminal, both models, `nar` commit, and the exact lines observed
  - _Evidence: <date, environment, observations>_
