# configuration Specification

## Purpose
Defines where `nar` reads its settings from, how user-level and project-level settings combine, which settings exist initially, and how invalid settings are reported.

## Requirements

### Requirement: Configuration file locations
`nar` SHALL read settings from a user-level file at `~/.nar/config.toml` and a project-level file at `.nar/config.toml` in the current working directory. Both files are optional and in TOML format. Both files SHALL be read as UTF-8; a UTF-8 byte order mark (BOM) at the start of a file SHALL be ignored, so a file saved with or without a BOM yields the same settings.

#### Scenario: No configuration files
- **WHEN** neither `~/.nar/config.toml` nor `.nar/config.toml` exists
- **THEN** `nar` starts using built-in default values and does not create any file

#### Scenario: Only the user-level file exists
- **WHEN** only `~/.nar/config.toml` exists
- **THEN** its values are used and any missing setting falls back to its default

#### Scenario: File saved with a UTF-8 BOM
- **WHEN** `.nar/config.toml` starts with a UTF-8 BOM followed by `approval = "yolo"` (as written by Windows PowerShell 5.1 `Set-Content -Encoding utf8`)
- **THEN** `nar` starts, and `nar --show-config` reports `approval = yolo` coming from `.nar/config.toml`

#### Scenario: File that is not valid UTF-8
- **WHEN** `.nar/config.toml` contains bytes that are not valid UTF-8
- **THEN** `nar` exits with code 1 and the error names the file

### Requirement: Project settings override user settings
When both files define the same setting, the project-level value SHALL win. Settings defined in only one file SHALL be kept. Nested tables SHALL be merged key by key rather than replaced as a whole.

#### Scenario: Same key in both files
- **WHEN** `~/.nar/config.toml` sets `model = "a"` and `.nar/config.toml` sets `model = "b"`
- **THEN** the effective model is `"b"`

#### Scenario: Nested tables merge
- **WHEN** the user file defines `[mcp_servers.jira]` and the project file defines `[mcp_servers.linear]`
- **THEN** the effective configuration contains both `jira` and `linear` servers

### Requirement: Initial settings
The configuration SHALL support at least these settings, each with a documented default:
- `provider`: which LLM provider to use; one of `anthropic`, `openrouter`, `ollama`; default `openrouter`.
- `model`: which model of that provider to use; when not set, the selected provider's documented default model is used.
- `approval`: how much autonomy the agent has; one of `ask`, `auto-edit`, `yolo`; default `ask`.
- `mcp_servers`: a table of named external MCP server definitions; default empty.

#### Scenario: Defaults applied
- **WHEN** no configuration file sets `approval`
- **THEN** the effective `approval` value is `ask`

#### Scenario: Effective configuration is inspectable
- **WHEN** the user runs `nar --show-config`
- **THEN** the effective merged configuration is printed, indicating which file (user, project or default) each value came from

#### Scenario: Model follows the provider
- **WHEN** a configuration file sets `provider = "ollama"` and no file sets `model`
- **THEN** the effective `model` is the documented default model for `ollama`

#### Scenario: Unsupported provider
- **WHEN** a configuration file sets `provider = "openai"`
- **THEN** `nar` exits with code 1 and the error names the `provider` setting and lists `anthropic, openrouter, ollama`

### Requirement: Configuration validation
`nar` SHALL validate the merged configuration at startup and refuse to start when it is invalid, reporting the file and the offending setting.

#### Scenario: Malformed TOML
- **WHEN** `.nar/config.toml` contains invalid TOML syntax
- **THEN** `nar` exits with code 1 and the error names the file and the line of the syntax error

#### Scenario: Invalid value
- **WHEN** a configuration file sets `approval = "always"`
- **THEN** `nar` exits with code 1 and the error names the `approval` setting and lists the allowed values

#### Scenario: Unknown setting
- **WHEN** a configuration file contains a setting `nar` does not recognize, such as `modle = "x"`
- **THEN** `nar` exits with code 1 and the error names the unknown setting and the file that contains it

### Requirement: Hook timeout setting

`nar` SHALL accept a `hook_timeout` setting: a whole or fractional number of seconds, at least 1 and at most 600, defaulting to `10`. It SHALL be the time each individual hook command gets before it is killed together with its child processes. `--show-config` SHALL show it with the file it came from, like every other setting.

#### Scenario: The default
- **WHEN** no file sets `hook_timeout`
- **THEN** `--show-config` shows `hook_timeout = 10` from the built-in defaults

#### Scenario: A project sets it
- **WHEN** the project file sets `hook_timeout = 30`
- **THEN** every hook gets 30 seconds and `--show-config` shows the project file as its source

#### Scenario: An out-of-range value
- **WHEN** the file sets `hook_timeout = 0` or `hook_timeout = 601`
- **THEN** `nar` refuses to start and the error names `hook_timeout` and the file

### Requirement: Command-line overrides
The `--provider` and `--model` command-line options SHALL override the `provider` and `model` values from every configuration file for that run, and SHALL be validated the same way. `nar --show-config` SHALL show them with the command line as their source.

#### Scenario: Flag beats project file
- **WHEN** `.nar/config.toml` sets `provider = "openrouter"` and the user runs `nar --provider ollama --show-config`
- **THEN** the effective provider is `ollama` and its source is shown as the command line

#### Scenario: Provider flag alone
- **WHEN** no configuration file sets `model` and the user runs `nar --provider anthropic --show-config`
- **THEN** the effective model is the documented default model for `anthropic`

### Requirement: Turn limit setting
The configuration SHALL support a `max_turns` setting: the maximum number of model calls the agent makes for one request. It SHALL be a whole number of at least 1, with default 20.

#### Scenario: Default turn limit
- **WHEN** no configuration file sets `max_turns`
- **THEN** the effective `max_turns` is 20 and `nar --show-config` shows it with source `default`

#### Scenario: Invalid turn limit
- **WHEN** a configuration file sets `max_turns = 0`
- **THEN** `nar` exits with code 1 and the error names the `max_turns` setting and the file
