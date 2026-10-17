# Design

## Context

The `nar` repo (https://github.com/rojb/nar) is empty. This change creates the package skeleton that every other change extends, so the choices here (layout, CLI shape, config model, error convention) become the conventions for the whole project. Motivation and scope: see `proposal.md`. Required behavior: see `specs/cli-entrypoint`, `specs/configuration` and `specs/project-instructions`.

Constraints:
- Python 3.11+ (team machines have 3.11.9), so `tomllib` is in the standard library.
- Must run on Windows (PowerShell) as well as Linux/macOS.
- `uv` is not installed on the team machines; the setup must work with plain `python -m venv` + `pip`.

## Goals / Non-Goals

**Goals:**
- A layout and module names that later changes plug into without moving files.
- One place (`NarConfig`) where every later change adds its settings.
- One error convention that every later change reuses.

**Non-Goals:**
- Rich rendering, prompt_toolkit input and slash commands (RRG-63, RRG-78). The interactive loop here is a plain `input()` placeholder.
- Any network or LLM access (RRG-61).
- Writing config files or a `nar init` command.

## Decisions

### 1. `src/` layout with hatchling
```
nar/                          (repo root)
├── pyproject.toml            NEW  hatchling build, entry point, ruff config, dev extras
├── README.md                 NEW  install + usage
├── .gitignore                NEW
└── src/nar/
    ├── __init__.py           NEW  __version__
    ├── __main__.py           NEW  `python -m nar` → cli.main
    ├── errors.py             NEW  NarError, ConfigError
    ├── cli/
    │   ├── __init__.py       NEW  re-exports main
    │   ├── main.py           NEW  Click command, error boundary
    │   └── repl.py           NEW  placeholder interactive loop
    ├── config/
    │   ├── __init__.py       NEW  re-exports load_config, NarConfig
    │   ├── paths.py          NEW  user/project config paths (NAR_HOME override)
    │   ├── models.py         NEW  NarConfig, ApprovalLevel (pydantic)
    │   └── loader.py         NEW  read TOML, deep-merge, validate, provenance
    └── context/
        ├── __init__.py       NEW
        └── instructions.py   NEW  ProjectInstructions, find_agents_md()
```
Entry point: `[project.scripts] nar = "nar.cli:main"`.
- **Why `src/`:** the repo and the package are both called `nar`; with a flat layout, running from the repo root imports the local folder instead of the installed package and hides packaging mistakes.
- **Why hatchling:** zero-config for `src/` layouts and supported by `pip install -e .`. Alternatives: setuptools (more boilerplate), poetry/uv (extra tool to install).

### 2. One Click command with an optional `PROMPT` argument
`nar [OPTIONS] [PROMPT]`: no prompt → `repl.run()`, prompt → single mode. Options in this change: `--version` (`click.version_option`), `--show-config`, `--help`.
- **Alternative rejected:** a Click group with subcommands (`nar chat`, `nar config show`). `nar "config"` would collide with a subcommand name and the video's UX is `nar` / `nar "prompt"`. Future flags (`--provider`, `--model`, `--approval`, `--resume`) are added as options on this same command.
- Single mode in this change prints a placeholder acknowledging the prompt; RRG-63 replaces it with the agent loop.

### 3. Error boundary in `cli.main`
`main()` wraps the Click command (`standalone_mode=False`) and:
- catches `NarError` → prints `error: <message>` in red to stderr via `click.secho(err=True)`, exits with `error.exit_code` (default 1);
- catches any other `Exception` → prints `error: unexpected failure (<type>): <message>` plus a hint to set `NAR_DEBUG=1`, exits 1;
- when `NAR_DEBUG=1`, prints the full traceback as well;
- lets Click's own usage errors keep Click's exit code 2.

`NarError(message: str, exit_code: int = 1)` is the base of every expected error; `ConfigError(NarError)` carries the file path. Later changes add subclasses (`ProviderError`, `ToolError`, …) instead of new boundaries.
- **Why not Rich now:** Rich arrives with RRG-63's renderer; adding it here for one red line would create a second styling path to reconcile later.

### 4. Configuration: dicts merged first, validated once
`load_config(cwd) -> LoadedConfig`:
1. Read `user = paths.user_config()` and `project = cwd / ".nar/config.toml"` with `tomllib` (missing file → `{}`). A `tomllib.TOMLDecodeError` becomes `ConfigError` with the file path and `lineno`.
2. Deep-merge `defaults ← user ← project` (dicts merge recursively, scalars and lists are replaced).
3. Record provenance: a flat map `"dotted.key" → "default" | <user path> | <project path>` built during the merge, used by `--show-config`.
4. Validate with `NarConfig.model_validate(merged)`; translate `pydantic.ValidationError` into `ConfigError` naming the setting, the allowed values (for enums) and the file the value came from (from provenance).

`NarConfig` (pydantic v2, `extra="forbid"` at the top level so typos like `modle` fail):

| Key | Type | Default |
|---|---|---|
| `provider` | `str` | `"openrouter"` |
| `model` | `str` | `"openrouter/auto"` |
| `approval` | `ApprovalLevel` = `ask` \| `auto-edit` \| `yolo` | `"ask"` |
| `mcp_servers` | `dict[str, dict[str, Any]]` | `{}` |

- `provider` stays a plain `str` here; RRG-61 narrows it to the providers it implements. `mcp_servers` entries stay loosely typed until RRG-74 defines their schema, so the strict unknown-key rule applies only to top-level keys for now.
- `NAR_HOME` env var overrides `~/.nar` (used for manual testing without touching the real home folder and, later, by sessions/memory).
- **Why merge before validating:** validating each file separately would reject a project file that only sets one key, and partial pydantic models duplicate every field. **Alternative rejected:** `pydantic-settings` (adds env-var/secret semantics we do not need yet and is harder to get per-key provenance from).

### 5. AGENTS.md discovery
`find_agents_md(cwd) -> Path | None`: check `cwd/AGENTS.md`; then walk parents until reaching a directory that contains `.git` (file or folder, so worktrees work) — that directory is checked last. If no `.git` is found on the way up, only `cwd` counts. `load_project_instructions(cwd) -> ProjectInstructions | None` reads at most 64 KB of bytes, decodes as UTF-8 (`UnicodeDecodeError`/`OSError` → warning + `None`), and sets `truncated=True` when the file was larger.

`ProjectInstructions` is a frozen dataclass `(path: Path, content: str, truncated: bool)`. Lives in `nar/context/` because RRG-63's `ContextManager` is its consumer. Warnings go through `click.secho(fg="yellow", err=True)` in this change.

## Risks / Trade-offs

- [Default `openrouter/auto` needs an OpenRouter key and may change] → It is only a default; RRG-61 owns provider defaults and can change them without touching these specs (the spec only requires a documented default).
- [`extra="forbid"` will break configs written for a newer `nar` version] → Acceptable for a course project; the error names the unknown key so the fix is obvious.
- [The `input()` placeholder REPL differs from the final prompt_toolkit UX] → It only needs to satisfy the exit/empty-input scenarios; RRG-78 swaps the input source behind `repl.run()`.
- [Windows end-of-input is Ctrl+Z+Enter, not Ctrl+D] → Both raise `EOFError` in `input()`; the spec scenario already names both.

## Migration Plan

New repo, nothing to migrate. Setup: `git clone https://github.com/rojb/nar` inside `nar-specs/`, `python -m venv .venv`, activate, `pip install -e ".[dev]"`. Rollback: uninstall with `pip uninstall nar`.
