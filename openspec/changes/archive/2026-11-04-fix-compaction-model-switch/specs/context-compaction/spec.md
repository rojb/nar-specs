# Spec Delta

## ADDED Requirements

### Requirement: Compaction uses the current provider
The summary of a compaction SHALL be requested from the provider and model selected at that moment, including after `/model`.

#### Scenario: Manual compaction after changing the model
- **WHEN** the user runs `/model` to change the provider and then `/compact`
- **THEN** the summary is requested from the new provider, and the previous one receives nothing

#### Scenario: Automatic compaction after changing the model
- **WHEN** the conversation goes over the context limit after `/model`
- **THEN** the summary is requested from the new provider
