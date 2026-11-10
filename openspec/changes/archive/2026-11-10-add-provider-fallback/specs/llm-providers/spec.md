# Spec Delta

## ADDED Requirements

### Requirement: A provider error says whether the provider cannot answer
A provider error SHALL say whether the provider cannot answer right now, as opposed to a mistake in the request or the credentials. It SHALL be marked as "cannot answer" in these cases:
- an account without balance (HTTP 402, or Anthropic's "credit balance is too low")
- rate limits, overload and server errors that are still failing after the retries
- a server that cannot be reached or does not answer in time

A rejected key, an unknown model and any other rejected request SHALL NOT be marked.

#### Scenario: No balance
- **WHEN** DeepSeek answers HTTP 402
- **THEN** the error says the balance is insufficient and is marked as "cannot answer"

#### Scenario: Retries exhausted
- **WHEN** a provider answers HTTP 429 or 529 on every attempt
- **THEN** the error after the last attempt is marked as "cannot answer"

#### Scenario: A rejected key is the user's to fix
- **WHEN** a provider answers HTTP 401
- **THEN** the error names the key variable and is not marked
