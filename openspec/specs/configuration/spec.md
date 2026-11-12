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
- `provider`: which LLM provider to use; one of `anthropic`, `openrouter`, `ollama`, `deepseek`; default `openrouter`.
- `model`: which model of that provider to use; when not set, the selected provider's documented default model is used.
- `approval`: how much autonomy the agent has; one of `ask`, `auto-edit`, `yolo`; default `ask`.
- `language`: which language the model replies in; one of `es`, `en`; default `es`.
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

#### Scenario: DeepSeek selected in a project file
- **WHEN** `.nar/config.toml` sets `provider = "deepseek"` and no file sets `model`
- **THEN** `nar --show-config` shows `provider = deepseek` with the project file as its source and `model` as the documented default model for `deepseek`

#### Scenario: Unsupported provider
- **WHEN** a configuration file sets `provider = "openai"`
- **THEN** `nar` exits with code 1 and the error names the `provider` setting and lists `anthropic, openrouter, ollama, deepseek`

#### Scenario: Language defaults to Spanish
- **WHEN** no configuration file and no option set `language`
- **THEN** `nar --show-config` shows `language = es` with `default` as its source

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

### Requirement: Context limit settings
The configuration SHALL support `context_window`, the size of the model's context in tokens, a whole number of at least 1000, whose default depends on the provider (`anthropic` 200000, `openrouter` 128000, `deepseek` 128000, `ollama` 4096), and `context_threshold`, the percentage of the window at which `nar` starts reducing the conversation, a whole number from 1 to 100, default 80.

#### Scenario: Default depends on the provider
- **WHEN** no file sets `context_window` and the provider is `ollama`
- **THEN** the effective `context_window` is 4096 and `--show-config` shows it

#### Scenario: Value from a file wins
- **WHEN** `.nar/config.toml` sets `context_window = 32000`
- **THEN** the effective value is 32000 with any provider

#### Scenario: Invalid threshold
- **WHEN** a file sets `context_threshold = 0`
- **THEN** `nar` stops with a one-line error naming the setting and the file, and exit code 1

### Requirement: Subagent turn limit
The configuration SHALL support `subagent_max_turns`, the model calls a subagent may make for one task, a whole number of at least 1, default 15.

#### Scenario: Default
- **WHEN** no file sets `subagent_max_turns`
- **THEN** the effective value is 15, and `--show-config` shows it

#### Scenario: Invalid value
- **WHEN** a file sets `subagent_max_turns = 0`
- **THEN** `nar` stops with a one-line error naming the setting and the file, and exit code 1

### Requirement: Approval level setting
The `approval` setting SHALL select the autonomy level of the agent: `ask` (every call of a tool that requires approval is asked about), `auto-edit` (file edits inside the working directory run on their own; everything else that requires approval is asked about) and `yolo` (nothing is asked). The default SHALL be `ask`. The `--approval` command-line option SHALL override the value from every configuration file for that run, SHALL be validated the same way, and `nar --show-config` SHALL show it with the command line as its source.

#### Scenario: Level from the project file
- **WHEN** `.nar/config.toml` sets `approval = "auto-edit"`
- **THEN** `nar --show-config` shows `approval = auto-edit` with `.nar/config.toml` as its source, and the agent edits files in the working directory without asking

#### Scenario: Flag beats file
- **WHEN** `.nar/config.toml` sets `approval = "ask"` and the user runs `nar --approval yolo --show-config`
- **THEN** the effective approval is `yolo` and its source is shown as the command line

#### Scenario: Default level is the safest
- **WHEN** no configuration file and no option set the approval level
- **THEN** the effective level is `ask` and the agent asks before every call of a tool that requires approval

### Requirement: MCP server definitions
Each entry of the `mcp_servers` table SHALL define one external MCP server, named by its table key (`[mcp_servers.<name>]`). A name SHALL start with a letter or digit, contain only letters, digits, `_` and `-`, and be at most 32 characters; two names that differ only by `-` versus `_` SHALL be rejected as ambiguous. An entry SHALL define exactly one transport: `command` (a program started by `nar`, with optional `args`, a list of strings, and `env`, a table of strings) or `url` (a streamable HTTP endpoint, with optional `headers`, a table of strings). Every entry MAY also set `enabled` (default `true`), `enabled_tools` (a list of the server's own tool names; default: all of the server's tools), `startup_timeout` (seconds to wait for the server to connect and list its tools; default 30) and `tool_timeout` (seconds to wait for one tool call; default 60). Both timeouts SHALL be positive numbers. Keys that belong to the other transport and unknown keys SHALL be rejected. Entries from the user file and the project file SHALL merge key by key like every other table. An invalid entry SHALL stop startup with an error that names the file, the server and the key.

#### Scenario: stdio server
- **WHEN** `.nar/config.toml` contains `[mcp_servers.files]` with `command = "npx"` and `args = ["-y", "@modelcontextprotocol/server-filesystem", "."]`
- **THEN** `nar --show-config` shows the server with `command`, `args`, the defaults `enabled = true`, `startup_timeout = 30` and `tool_timeout = 60`, and `.nar/config.toml` as the source

#### Scenario: HTTP server
- **WHEN** a file contains `[mcp_servers.linear]` with `url = "https://mcp.linear.app/mcp"` and `headers = { Authorization = "Bearer ${LINEAR_API_KEY}" }`
- **THEN** the configuration is valid and `nar` starts

#### Scenario: Both transports in one entry
- **WHEN** an entry sets both `command` and `url`
- **THEN** `nar` exits with code 1 and the error names the file and the server and says that exactly one of `command` or `url` must be set

#### Scenario: Neither transport
- **WHEN** an entry sets neither `command` nor `url`
- **THEN** `nar` exits with code 1 and the error names the server and says that exactly one of `command` or `url` must be set

#### Scenario: Key of the other transport
- **WHEN** an entry with `url` also sets `args`
- **THEN** `nar` exits with code 1 and the error names the server and the key `args`

#### Scenario: Unknown key in an entry
- **WHEN** an entry contains `comand = "npx"`
- **THEN** `nar` exits with code 1 and the error names the unknown setting `mcp_servers.<name>.comand` and the file

#### Scenario: Invalid timeout
- **WHEN** an entry sets `tool_timeout = 0`
- **THEN** `nar` exits with code 1 and the error names the server and `tool_timeout`

#### Scenario: Invalid or ambiguous name
- **WHEN** the configuration defines servers named `my-srv` and `my_srv`, or a server named `jira server`
- **THEN** `nar` exits with code 1 and the error names the offending name

#### Scenario: Servers from both files
- **WHEN** the user file defines `[mcp_servers.jira]` and the project file defines `[mcp_servers.linear]` and also sets `enabled = false` for `jira`
- **THEN** both servers are in the effective configuration and `jira` is disabled

### Requirement: Secrets are not printed by --show-config
`nar --show-config` SHALL NOT print a literal secret from an MCP server definition: the value of every `env` entry and every `headers` entry SHALL be shown as `********` unless the value contains a `${NAME}` reference, in which case it SHALL be shown exactly as written (so the user can see which variable is used without exposing its value). The same masking SHALL apply wherever `nar` prints the effective configuration. The values of environment variables SHALL never be printed.

#### Scenario: Reference is shown, literal is masked
- **WHEN** a server sets `headers = { Authorization = "Bearer ${LINEAR_API_KEY}", X-Team = "abc123" }` and the user runs `nar --show-config`
- **THEN** the `Authorization` header is shown as `Bearer ${LINEAR_API_KEY}` and the `X-Team` header is shown as `********`

#### Scenario: Environment value never shown
- **WHEN** `LINEAR_API_KEY` is set in the environment and `nar --show-config` is run
- **THEN** the value of `LINEAR_API_KEY` does not appear in the output

### Requirement: Language setting
The `language` setting SHALL select the language the model replies to the user in: `es` (Spanish) or `en` (English). The default SHALL be `es`. It SHALL be read from `~/.nar/config.toml` and `.nar/config.toml` like the other settings, with the project value winning. The `--language` command-line option SHALL override the value from every configuration file for that run, SHALL be validated the same way, and `nar --show-config` and `/config` SHALL show it with its source, the command line included. The language SHALL NOT change during a session.

#### Scenario: Language from the user file
- **WHEN** `~/.nar/config.toml` sets `language = "en"` and no other file or option sets it
- **THEN** `nar --show-config` shows `language = en` with the user file as its source

#### Scenario: Project file beats user file
- **WHEN** `~/.nar/config.toml` sets `language = "en"` and `.nar/config.toml` sets `language = "es"`
- **THEN** the effective language is `es` and its source is the project file

#### Scenario: Flag beats file
- **WHEN** `.nar/config.toml` sets `language = "es"` and the user runs `nar --language en --show-config`
- **THEN** the effective language is `en` and its source is shown as the command line

#### Scenario: Language in /config
- **WHEN** the user types `/config` in interactive mode
- **THEN** `language` is listed with its value and its source, like every other setting

#### Scenario: Unsupported language in a file
- **WHEN** a configuration file sets `language = "fr"`
- **THEN** `nar` exits with code 1 and the error names the `language` setting and lists `es, en`
