# Spec Delta

## ADDED Requirements

### Requirement: Summary language
The instructions given to the model for a summary SHALL tell it to write the summary in the configured language, keeping file paths, names, commands and values exact. A subagent SHALL summarize in the same language as the main agent.

#### Scenario: Summary in Spanish
- **WHEN** the language is `es` and the conversation is compacted
- **THEN** the instructions for the summary tell the model to write it in Spanish

#### Scenario: Summary in English
- **WHEN** the language is `en` and the user types `/compact`
- **THEN** the instructions for the summary tell the model to write it in English
