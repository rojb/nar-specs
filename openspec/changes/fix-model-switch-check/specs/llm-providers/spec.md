# Spec Delta

## ADDED Requirements

### Requirement: A provider can check its access without a model call
Every provider SHALL be able to check, without sending a conversation or spending tokens, that its server answers, that its key is accepted and that its model exists. The check SHALL make a single attempt with a short timeout, and SHALL fail with the same one-line messages a request uses. When the model does not exist and the provider lists its models, the error SHALL suggest up to three close names. For Ollama, a model that is not installed SHALL suggest `ollama pull <model>`, and a name without a tag SHALL match the `latest` tag.

#### Scenario: A rejected key
- **WHEN** a provider is checked with a key its server rejects
- **THEN** the check fails with the message that names the key's environment variable

#### Scenario: A model that does not exist
- **WHEN** a provider is checked with a misspelled model name such as `deepseek-flsh`
- **THEN** the check fails with `model 'deepseek-flsh' was not found by deepseek` and suggests `deepseek-flash`

#### Scenario: Ollama is not running
- **WHEN** Ollama is checked and nothing listens at its host
- **THEN** the check fails once, without retries, with the message that says how to start it

#### Scenario: An Ollama model that is not installed
- **WHEN** Ollama is checked with a model it has not pulled
- **THEN** the check fails with a message that suggests `ollama pull <model>`

#### Scenario: Everything answers
- **WHEN** the server answers, the key is accepted and the model is listed
- **THEN** the check passes and no model call is made
