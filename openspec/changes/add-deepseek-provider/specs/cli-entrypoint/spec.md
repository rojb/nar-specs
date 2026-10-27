# Spec Delta

## MODIFIED Requirements

### Requirement: Provider and model options
The `nar` command SHALL accept `--provider <name>` and `--model <name>` options in both modes, choosing the provider and model for that run; `--help` SHALL list them together with the allowed provider names (`anthropic`, `openrouter`, `ollama`, `deepseek`).

#### Scenario: Choose provider and model from the command line
- **WHEN** the user runs `nar --provider ollama --model qwen2.5-coder "hello"`
- **THEN** the prompt is answered by the `qwen2.5-coder` model of the local Ollama server, whatever the configuration files say

#### Scenario: Choose DeepSeek from the command line
- **WHEN** the user runs `nar --provider deepseek "hello"` with `DEEPSEEK_API_KEY` set
- **THEN** the prompt is answered by the documented default model of DeepSeek, whatever the configuration files say

#### Scenario: Help lists DeepSeek
- **WHEN** the user runs `nar --help`
- **THEN** the `--provider` option lists `deepseek` among the allowed names

#### Scenario: Invalid provider option
- **WHEN** the user runs `nar --provider openai "hello"`
- **THEN** `nar` exits with a non-zero code and the error lists `anthropic`, `openrouter`, `ollama` and `deepseek`
