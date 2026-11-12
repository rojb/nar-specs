# Spec Delta

## ADDED Requirements

### Requirement: Subagent reply language
The system prompt of every subagent SHALL tell it to reply in the configured language, with the same exceptions as the main system prompt: code, identifiers, file names, paths, commands, tool arguments and quoted text stay unchanged.

#### Scenario: Subagent told to reply in Spanish
- **WHEN** the language is `es` and the model calls `task`
- **THEN** the first model call of the subagent has a system prompt that tells it to reply in Spanish

#### Scenario: Subagent told to reply in English
- **WHEN** the language is `en` and the model calls `task`
- **THEN** the first model call of the subagent has a system prompt that tells it to reply in English
