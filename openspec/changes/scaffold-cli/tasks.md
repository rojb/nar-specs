# Tasks

## [specs]

- [ ] 0.1 Link this change to Linear RRG-60 (move the issue to In Progress and add the `nar-specs` change path as a link) and verify the link opens `openspec/changes/scaffold-cli/` on GitHub

## [nar]

## 1. Repository and packaging

- [ ] 1.1 Clone https://github.com/rojb/nar inside the `nar-specs` folder as `nar/` and create a branch from Linear's RRG-60 branch name; verify `git status` in `nar/` shows the new branch and `git status` in `nar-specs` does not list `nar/`
- [ ] 1.2 Add `pyproject.toml` (hatchling, `requires-python = ">=3.11"`, dependencies `click` and `pydantic>=2`, `[project.optional-dependencies] dev = ["ruff", "pytest"]`, `[project.scripts] nar = "nar.cli:main"`, ruff settings) and `.gitignore` (`.venv/`, `__pycache__/`, `*.egg-info/`, `.pytest_cache/`, `.ruff_cache/`); verify `python -m venv .venv` + `pip install -e ".[dev]"` succeeds
- [ ] 1.3 Create the `src/nar/` skeleton from design.md §1 (`__init__.py` with `__version__ = "0.1.0"`, `__main__.py`, empty `cli/`, `config/`, `context/` packages); verify `python -c "import nar; print(nar.__version__)"` prints `0.1.0`

## 2. Errors and CLI entry point

- [ ] 2.1 Implement `nar/errors.py` (`NarError(message, exit_code=1)`, `ConfigError(NarError)` with the file path) and add `tests/test_errors.py`; verify `pytest tests/test_errors.py` passes
- [ ] 2.2 Implement `nar/cli/main.py`: one Click command `nar [OPTIONS] [PROMPT]` with `--version` and `--help`; a prompt prints a single-mode placeholder and exits 0. Verify `nar --version` from a folder outside the repo prints `nar, version 0.1.0` and `nar "hola"` prints the placeholder and exits 0
- [ ] 2.3 Implement the error boundary in `main()` (NarError → red one-line stderr + its exit code; other exceptions → generic message + `NAR_DEBUG=1` hint, exit 1; full traceback when `NAR_DEBUG=1`) and add `tests/test_cli.py` using Click's `CliRunner` for `--version`, `--help`, single mode and a forced `NarError`; verify `pytest tests/test_cli.py` passes
- [ ] 2.4 Implement `nar/cli/repl.py` placeholder loop (`input()`; ignores empty lines; `exit`, EOF and Ctrl+C at the prompt end it with code 0) and wire `nar` with no prompt to it; verify manually on Windows: start `nar`, press Enter on an empty line, then Ctrl+Z+Enter — no traceback and `$LASTEXITCODE` is 0

## 3. Configuration

- [ ] 3.1 Implement `nar/config/paths.py` (`user_config()` honoring `NAR_HOME`, `project_config(cwd)`) and `nar/config/models.py` (`ApprovalLevel`, `NarConfig` with the defaults table from design.md §4, `extra="forbid"`); verify `NarConfig()` returns the documented defaults in a pytest case
- [ ] 3.2 Implement `nar/config/loader.py` (`load_config(cwd)`: read TOML, deep-merge defaults ← user ← project, record provenance, validate, translate `TOMLDecodeError`/`ValidationError` into `ConfigError` with file, line or setting, and allowed values) and add `tests/test_config.py` covering every scenario in `specs/configuration/spec.md` with `NAR_HOME` and `tmp_path`; verify `pytest tests/test_config.py` passes
- [ ] 3.3 Load the configuration at CLI startup and add `--show-config` (prints each effective key with its source: `default`, user path or project path); verify `nar --show-config` with a project file setting `model = "b"` shows `model = b  (.nar/config.toml)` and an invalid `approval = "always"` makes `nar` exit 1 with a one-line error listing `ask, auto-edit, yolo`

## 4. Project instructions

- [ ] 4.1 Implement `nar/context/instructions.py` (`ProjectInstructions` frozen dataclass, `find_agents_md(cwd)` walking up to the directory containing `.git`, `load_project_instructions(cwd)` with the 64 KB limit, UTF-8 decode and warnings) and add `tests/test_instructions.py` covering every scenario in `specs/project-instructions/spec.md`; verify `pytest tests/test_instructions.py` passes
- [ ] 4.2 Load project instructions at startup and show `Using AGENTS.md: <path>` when interactive mode starts; verify by running `nar` from a subfolder of a repo whose root has `AGENTS.md`

## 5. Documentation and delivery

- [ ] 5.1 Write `nar/README.md` (requirements, venv + `pip install -e ".[dev]"`, `nar` / `nar "prompt"`, config file locations, keys and defaults, `--show-config`, `NAR_HOME`, `NAR_DEBUG`, `AGENTS.md`); verify every command in the README runs as written on a fresh venv
- [ ] 5.2 Run `ruff check .`, `ruff format --check .` and `pytest`; verify all three pass, then push the branch and open a PR referencing RRG-60

## 6. Manual verification (RRG-60 "done when")

- [ ] 6.1 Walk through RRG-60's "done when" in a fresh terminal and record what was observed for each step:
  - `nar --version` from a folder outside the repo → prints the version, exit 0
  - `nar --help` from that folder → shows interactive and single modes, exit 0
  - `nar --show-config` with no config files → shows defaults (`provider = openrouter`, `model = openrouter/auto`, `approval = ask`)
  - Add `~/.nar/config.toml` (or `NAR_HOME`) and `.nar/config.toml` with an overlapping key → project value wins and the source is shown
  - Break `.nar/config.toml` (bad TOML, invalid `approval`, unknown key `modle`) → one-line error, exit 1, no traceback; with `NAR_DEBUG=1` the traceback appears
  - `nar` in interactive mode → shows `Using AGENTS.md: …` when present, `exit` ends with code 0
