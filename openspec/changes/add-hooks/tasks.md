# Tasks

> Review workload: about 350 lines of code and docs plus 450 lines of tests, over the ~400-line
> guide (a planning heuristic, not a cap; tests are never dropped to fit). Split into two PRs:
> PR A = groups 2 and 3 (config and the runner, not wired to the agent), PR B = groups 4 to 6 (the
> loop, the display, and the docs), each retargeted to `main` before its parent merges. The branch
> name and the "done when" are the ones the Linear issue shows, read on 2026-11-04.

## [specs]

- [ ] 0.1 Link this change to Linear RRG-72 (move the issue from Backlog to In Progress and add the `nar-specs` change path as a link on the issue); verify the link opens `openspec/changes/add-hooks/` on GitHub once merged
- [ ] 0.2 Open a PR in `nar-specs` from Linear's RRG-72 branch with proposal, specs, design and tasks; verify `openspec validate add-hooks --strict` passes

## [nar]

## 1. Branch

- [ ] 1.1 In `nar/`, update `main` and create `rubenjustiniano96/rrg-72-add-hooks-hooks-antes-y-despues-de-cada-tool-y-del-turno` (the `gitBranchName` of RRG-72); verify a clean tree, that `pip install -e ".[dev]"` works, and that `pytest -q` and `ruff check .` pass before any change _Evidence: baseline on `main` at `7268b23`: `pytest -q` 1060 passed, 1 skipped; `ruff check .` clean; `ruff format --check .` 128 files already formatted._

## 2. Configuration (PR A)

- [ ] 2.1 In `src/nar/config/models.py` add `HookEvent` (`StrEnum`: `BEFORE_AGENT`, `BEFORE_TOOL`, `AFTER_TOOL`, `AFTER_AGENT`, `ON_ERROR`), `hooks: dict[HookEvent, list[str]]` with a field validator that rejects an unknown event name as `unknown setting 'hooks.<name>'`, and `hook_timeout: float = Field(default=10, ge=1, le=600)`; extend `tests/test_config.py`; verify `pytest tests/test_config.py` passes with every existing test unchanged and for: a project file with two events loads both lists in order, an unknown key `before_tolls` fails naming `hooks.before_tolls` and the file, a value that is a string instead of a list fails, `hook_timeout = 0` and `601` fail naming the key, the user file and the project file merge event by event, `--show-config` shows `hook_timeout` with its source and does not mask hook commands, and `python -c "import nar.config, sys; assert 'nar.hooks' not in sys.modules"` exits 0

## 3. The runner (PR A)

- [ ] 3.1 Create `src/nar/hooks/runner.py` with `HookRunner` (design.md §3 to §5), `payload_for(event, ...)`, `run_hook(...)` reusing `nar.tools.shell_process`, and `src/nar/hooks/__init__.py` re-exporting them, plus `tests/test_hooks_runner.py`; verify `pytest tests/test_hooks_runner.py` passes for: no hooks configured runs no process (checked with a monkeypatched `run_hook` that fails if called) and returns `None`, each event builds the payload of design.md §3 with `cwd` and a newline at the end, a `before_tool` hook exiting 0 returns `None` and the tool runs, one exiting 1 returns a `ToolResult` with `is_error` true, `error_kind` `REFUSED` and the hook's stdout and stderr in the output, a missing command warns once, returns `None` and does not block, a hook that exceeds `hook_timeout` is killed with its children and warns with `timed out`, a non-zero exit on `after_tool` warns and changes nothing, `after_tool` output is cut at 2,000 characters in the line shown to the user and `output` in the payload at 4,000, a hook that writes nothing produces no note, and the payload of a tool result with non-ASCII text survives
## 4. The loop (PR B)

- [ ] 4.1 Add `HookRan`, `HookBlocked` and `HookFailed` to `src/nar/agent/events.py`, `Agent(hooks=None)` and the call sites of design.md §6 to `src/nar/agent/loop.py`, and build the runner in `src/nar/cli/runner.py`; extend `tests/test_agent_loop.py` and `tests/test_cli.py`; verify `pytest tests/test_agent_loop.py tests/test_cli.py` passes for: an agent without hooks behaves exactly as before (every existing test unchanged), `before_agent` runs once before the first model call, `before_tool` runs after approval and never for a call the user rejected, a blocking hook leaves the file untouched and the model receives its text as the tool result with no failure envelope, `after_tool` runs with the result and cannot change it, `on_error` runs for a failing tool and for a provider error, `after_agent` runs with `finished`, `turn_limit`, `loop` and `cancelled` for each of those endings including Ctrl+C, and a hook that fails leaves the request working end to end through `nar`
## 5. Display (PR B)

- [ ] 5.1 Show the three hook events in `src/nar/ui/agent_view.py` on standard error, printed literally; extend `tests/test_agent_view.py`; verify `pytest tests/test_agent_view.py` passes for: a hook line naming the event and the command, its output as dim lines, nothing on an empty output, a refusal warning naming the first line of the hook's output, a failure warning, `[red]` in a command printed literally, and nothing on standard output

## 6. Docs and checks (PR B)

- [ ] 6.1 Update `nar/README.md` with a `Hooks` section: the five events, the `[hooks]` and `hook_timeout` settings, the JSON each event carries, that a `before_tool` hook blocks on a non-zero exit, that hooks run synchronously and fail open, and the worked example of `ruff` after every `edit_file` on a `.py` (Git Bash and PowerShell variants); verify every command in the section runs as written from the repo root, including the example hook firing on a real edit and its output appearing on standard error
- [ ] 6.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass with the only skip being the one recorded in 1.1 _Evidence: <commit>_

## 7. Delivery

- [ ] 7.1 For PR A and PR B: run `ruff check .`, `ruff format --check .` and `pytest`; commit with Conventional Commit messages (for example `feat(config): add the hooks table (RRG-72)`, `feat(hooks): run shell hooks around tool calls (RRG-72)`), push and open the PRs referencing RRG-72, retargeting PR B to `main` before merging PR A
- [ ] 7.2 Merge both PRs and confirm `main` contains the branch with `pytest` and `ruff check .` clean _Evidence: <merge commits>_

## 8. Manual verification (RRG-72 "done when")

- [ ] 8.1 Walk through RRG-72's "done when" ("el hook de ejemplo corre automáticamente tras una edición y su salida se ve en la TUI": the example hook runs by itself after an edit and its output is visible in the TUI) at the `nar/` repo root with `NAR_HOME` pointing at a temp folder and a temp project whose `.nar/config.toml` holds the README example, and record what was observed for each step:
  - Hook: with `ruff` on the PATH and `[hooks]` holding `after_tool = ["ruff check --fix ."]`, ask `nar` to create or edit a `.py` file and record the `hook after_tool: ...` line and ruff's output on standard error, and that standard output kept only the answer
  - Blocking: add a `before_tool` hook that exits 1 with a message for `write_file`, ask `nar` to write a file, and record that the file was not created, the warning line, and the model's answer reacting to the message
  - Isolation: break the hook (a command that does not exist), run a request, and record the warning and that the request still finished
  - Events: configure a hook that appends its JSON to a file for each of the five events, run one request that edits a file and fails a tool, then press Ctrl+C in another, and record the five payloads
  - Setting: `hook_timeout = 1` with a hook that sleeps 3 seconds; record the timeout warning and that the request continued
  - Recording: date, OS and shell, provider and model, `nar` commit, and the exact lines observed
  - _Evidence: <date, environment, observations>_
