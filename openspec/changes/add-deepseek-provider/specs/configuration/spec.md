# Spec Delta

## MODIFIED Requirements

### Requirement: Initial settings
The configuration SHALL support at least these settings, each with a documented default:
- `provider`: which LLM provider to use; one of `anthropic`, `openrouter`, `ollama`, `deepseek`; default `openrouter`.
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

#### Scenario: DeepSeek selected in a project file
- **WHEN** `.nar/config.toml` sets `provider = "deepseek"` and no file sets `model`
- **THEN** `nar --show-config` shows `provider = deepseek` with the project file as its source and `model` as the documented default model for `deepseek`

#### Scenario: Unsupported provider
- **WHEN** a configuration file sets `provider = "openai"`
- **THEN** `nar` exits with code 1 and the error names the `provider` setting and lists `anthropic, openrouter, ollama, deepseek`
