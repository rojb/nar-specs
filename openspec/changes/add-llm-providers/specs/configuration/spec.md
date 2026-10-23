# Spec Delta

## MODIFIED Requirements

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

## ADDED Requirements

### Requirement: Command-line overrides
The `--provider` and `--model` command-line options SHALL override the `provider` and `model` values from every configuration file for that run, and SHALL be validated the same way. `nar --show-config` SHALL show them with the command line as their source.

#### Scenario: Flag beats project file
- **WHEN** `.nar/config.toml` sets `provider = "openrouter"` and the user runs `nar --provider ollama --show-config`
- **THEN** the effective provider is `ollama` and its source is shown as the command line

#### Scenario: Provider flag alone
- **WHEN** no configuration file sets `model` and the user runs `nar --provider anthropic --show-config`
- **THEN** the effective model is the documented default model for `anthropic`
