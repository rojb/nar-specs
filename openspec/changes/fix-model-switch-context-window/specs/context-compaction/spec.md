# Spec Delta

## ADDED Requirements

### Requirement: The context limit follows the provider in use
When the provider or model changes during a session, with `/model` or by picking another provider after a failure, the context window SHALL become the new provider's default, unless `context_window` was set in a configuration file or on the command line. `context_threshold` SHALL stay as it was. If the conversation is over the new limit, it SHALL be reduced before the next model call, as for any conversation over the limit. Subagents started after the switch SHALL use the new limit.

#### Scenario: From Ollama to DeepSeek
- **WHEN** a session starts on Ollama with no `context_window` set and the user types `/model deepseek`
- **THEN** the context usage shows `/ 128,000 tokens`

#### Scenario: From a large window to a small one
- **WHEN** the conversation uses 20,000 tokens on DeepSeek and the user switches to Ollama
- **THEN** the window becomes 4,096 tokens, and the conversation is reduced before the next request reaches Ollama

#### Scenario: A window set by the user
- **WHEN** `.nar/config.toml` sets `context_window = 32000` and the user switches provider
- **THEN** the window stays 32,000 tokens
