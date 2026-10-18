# configuration Specification

## Purpose
Defines where `nar` reads its settings from, how user-level and project-level settings combine, which settings exist initially, and how invalid settings are reported.

## Requirements

### Requirement: Configuration file locations
`nar` SHALL read settings from a user-level file at `~/.nar/config.toml` and a project-level file at `.nar/config.toml` in the current working directory. Both files are optional and in TOML format.

#### Scenario: No configuration files
- **WHEN** neither `~/.nar/config.toml` nor `.nar/config.toml` exists
- **THEN** `nar` starts using built-in default values and does not create any file

#### Scenario: Only the user-level file exists
- **WHEN** only `~/.nar/config.toml` exists
- **THEN** its values are used and any missing setting falls back to its default

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
- `provider`: which LLM provider to use.
- `model`: which model of that provider to use.
- `approval`: how much autonomy the agent has; one of `ask`, `auto-edit`, `yolo`; default `ask`.
- `mcp_servers`: a table of named external MCP server definitions; default empty.

#### Scenario: Defaults applied
- **WHEN** no configuration file sets `approval`
- **THEN** the effective `approval` value is `ask`

#### Scenario: Effective configuration is inspectable
- **WHEN** the user runs `nar --show-config`
- **THEN** the effective merged configuration is printed, indicating which file (user, project or default) each value came from

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
