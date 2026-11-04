# Spec Delta

## ADDED Requirements

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
