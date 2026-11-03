# Spec Delta

## ADDED Requirements

### Requirement: Context limit settings
The configuration SHALL support `context_window`, the size of the model's context in tokens, a whole number of at least 1000, whose default depends on the provider (`anthropic` 200000, `openrouter` 128000, `deepseek` 128000, `ollama` 8192), and `context_threshold`, the percentage of the window at which `nar` starts reducing the conversation, a whole number from 1 to 100, default 80.

#### Scenario: Default depends on the provider
- **WHEN** no file sets `context_window` and the provider is `ollama`
- **THEN** the effective `context_window` is 8192 and `--show-config` shows it

#### Scenario: Value from a file wins
- **WHEN** `.nar/config.toml` sets `context_window = 32000`
- **THEN** the effective value is 32000 with any provider

#### Scenario: Invalid threshold
- **WHEN** a file sets `context_threshold = 0`
- **THEN** `nar` stops with a one-line error naming the setting and the file, and exit code 1
