# Spec Delta

## ADDED Requirements

### Requirement: Slash commands in interactive mode
In interactive mode, a line that starts with `/` followed by a command name (a letter, then letters, digits or `-`) and then either nothing or whitespace SHALL be handled as a command by `nar` and SHALL NOT be sent to the model or added to the conversation. Text after the name is the command's argument. A line that starts with `/` but is not of that form (for example a file path such as `/etc/hosts`) SHALL be treated as an ordinary request. A command name `nar` does not know SHALL print an error that lists the available commands and the session SHALL continue. Commands SHALL NOT be available in single mode, where the prompt is always a request. The available commands in this change are `/mcp`; further commands are added by later changes.

#### Scenario: Known command
- **WHEN** the user types `/mcp` at the `nar>` prompt
- **THEN** `nar` prints the MCP servers and returns to the prompt without calling the model

#### Scenario: Unknown command
- **WHEN** the user types `/foo` at the prompt
- **THEN** `nar` prints an error that `/foo` is not a command and lists the available commands, does not call the model, and shows the prompt again

#### Scenario: Path is not a command
- **WHEN** the user types `/etc/hosts what is this file?`
- **THEN** the line is sent to the model as a request

#### Scenario: Single mode
- **WHEN** the user runs `nar "/mcp"`
- **THEN** the text `/mcp` is sent to the model as the prompt
