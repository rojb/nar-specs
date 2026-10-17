# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-60
- **Video timestamps covered:** 00:01:28 (Project setup & Virtual Environment), 01:07:59 (Converting to CLI application using Click), 06:29:22 (Configuration System & Error Handling + AGENTS.md support)
- **Depends on:** none
- **Unblocks:** RRG-61 `add-llm-providers`, RRG-62 `add-tool-framework` (and transitively every other change)
- **Course requirement covered:** base for *selectable model provider* (provider/model config keys; the providers themselves are RRG-61)

## Why

`nar` does not exist yet: there is no installable package, no `nar` command and no place to read settings from. Every other change in the roadmap needs a stable entry point, a configuration model and a shared error convention to build on, so this foundation has to land first.

## What Changes

- Create the `nar` Python package (Python ≥ 3.11) with a `pyproject.toml` that exposes the console command `nar` (entry point `nar = nar.cli:main`), installable with `pip install -e .` inside a virtual environment, with ruff configured for lint and format.
- Add a Click-based CLI:
  - `nar` with no arguments starts **interactive mode** (placeholder REPL until RRG-63 adds the agent loop).
  - `nar "<prompt>"` runs **single mode** (placeholder that echoes the prompt until RRG-63).
  - `nar --version` and `nar --help`.
- Add a **configuration system**: a global file `~/.nar/config.toml` merged with a project file `.nar/config.toml` (project values override global ones), validated on load, with defaults when no file exists. Initial keys: LLM provider and model, approval level, and an `mcp_servers` table reserved for RRG-74. `nar --show-config` prints the effective configuration and where each value came from.
- Add **AGENTS.md discovery**: locate `AGENTS.md` at the project root and expose its contents as project instructions for the agent loop (consumed by RRG-63).
- Add a **base error hierarchy** (a root `NarError` and a `ConfigError` for invalid configuration) and make the CLI show these errors as a clear one-line message with a non-zero exit code instead of a traceback.

Out of scope: LLM calls (RRG-61), tools (RRG-62), the agent loop and Rich streaming output (RRG-63), approval behavior (RRG-69), MCP connections (RRG-74). Flags like `--provider`, `--model`, `--approval` and `--resume` are added by the changes that implement them.

## Capabilities

### New Capabilities

- `cli-entrypoint`: the `nar` command, its interactive and single modes, `--version`/`--help`, and how errors are reported to the user.
- `configuration`: where configuration files live, how global and project files merge, defaults, validation and the initial set of keys.
- `project-instructions`: discovery and loading of `AGENTS.md` as project-level instructions.

### Modified Capabilities

<!-- None: the store has no existing specs. -->

## Impact

- **New repo content:** `nar` (https://github.com/rojb/nar): `pyproject.toml`, `nar/` package with `cli` and `config` subpackages and the error module, `README.md`, `.gitignore`, ruff config.
- **Dependencies:** `click`, `pydantic` (config validation); `tomllib` from the standard library to read TOML. `rich` may be added now for error display or deferred to RRG-63 (decided in design).
- **User filesystem:** reads `~/.nar/config.toml` and `.nar/config.toml`; writes nothing yet.
- **Downstream:** RRG-61 reads the provider/model keys, RRG-63 consumes the AGENTS.md loader and the CLI modes, RRG-69 reads the approval level, RRG-74 reads `mcp_servers`.
