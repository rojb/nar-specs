# Spec Delta

## ADDED Requirements

### Requirement: /model checks the new provider before switching
Before switching, `/model` SHALL check the new provider and model. While it checks, it SHALL say so on standard error. When the check fails, `/model` SHALL print one error line that ends by naming the provider and model still in use, and the agent, compaction and the next request SHALL keep the previous provider. Ctrl+C during the check SHALL cancel the switch with the same guarantee, and the session SHALL continue.

#### Scenario: Ollama is stopped
- **WHEN** the user types `/model ollama` and Ollama is not running, then asks a question
- **THEN** one error line says Ollama cannot be reached and that the previous provider is still in use, and the question is answered by the previous provider

#### Scenario: An invalid key
- **WHEN** the user types `/model deepseek` with a `DEEPSEEK_API_KEY` that DeepSeek rejects
- **THEN** one error line names `DEEPSEEK_API_KEY`, and nothing changes

#### Scenario: A misspelled model
- **WHEN** the user types `/model anthropic claude-sonet-5-5`
- **THEN** the error says the model was not found, suggests close names, and nothing changes

#### Scenario: Cancelling the check
- **WHEN** the user presses Ctrl+C while `/model` is checking
- **THEN** the switch is cancelled, the previous provider stays and the prompt comes back

#### Scenario: A successful switch
- **WHEN** the check passes
- **THEN** `/model` switches and reports the new provider and model, as before
