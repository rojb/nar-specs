# Spec Delta

## ADDED Requirements

### Requirement: A provider can list its models
Every provider that its server allows SHALL be able to return the names of its models, without sending a conversation or spending tokens. The request SHALL make a single attempt with the same short timeout as the access check, and SHALL fail with the same one-line messages. A provider that cannot list its models SHALL say so instead of returning an empty list.

#### Scenario: Listing DeepSeek's models
- **WHEN** DeepSeek's models are listed with a valid key
- **THEN** the names the server reports are returned, and no model call is made

#### Scenario: A rejected key
- **WHEN** the models are listed with a key the server rejects
- **THEN** the request fails with the message that names the key's environment variable

#### Scenario: Ollama is not running
- **WHEN** Ollama's models are listed and nothing listens at its host
- **THEN** the request fails once, without retries, with the message that says how to start it
