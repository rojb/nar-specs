# Spec Delta

## MODIFIED Requirements

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

## ADDED Requirements

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
