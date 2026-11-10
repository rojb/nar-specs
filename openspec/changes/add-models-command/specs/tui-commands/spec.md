# Spec Delta

## ADDED Requirements

### Requirement: /models lists a provider's models
`/models` SHALL list the models of the provider in use, sorted by name, one per line on standard error, and SHALL mark the model in use. `/models <provider>` SHALL list that provider's models without switching, and an unknown provider SHALL be rejected with the list of providers. `/models <provider> <text>` SHALL keep only the names that contain the text, ignoring case, and SHALL say so when none match. A missing key, a rejected key or a server that does not answer SHALL print one error line, and the session SHALL continue. Ctrl+C while listing SHALL cancel the listing only.

#### Scenario: The provider in use
- **WHEN** DeepSeek is in use with `deepseek-flash`, and the user types `/models`
- **THEN** DeepSeek's models are listed, `deepseek-flash` is marked as in use, and the provider stays the same

#### Scenario: Another provider
- **WHEN** the user types `/models anthropic` with `ANTHROPIC_API_KEY` set
- **THEN** Anthropic's models are listed, and the provider in use does not change

#### Scenario: Filtering
- **WHEN** the user types `/models openrouter claude`
- **THEN** only OpenRouter's models whose names contain `claude` are listed, with how many there are

#### Scenario: No match
- **WHEN** the filter matches no name
- **THEN** one line says that no model matches, and how many models the provider has

#### Scenario: A missing key
- **WHEN** the user types `/models anthropic` without `ANTHROPIC_API_KEY`
- **THEN** one error line names `ANTHROPIC_API_KEY`, and the session continues
