# Spec Delta

## ADDED Requirements

### Requirement: Another provider is offered when the current one cannot answer
In interactive mode, with both standard input and standard error at a terminal, when a request fails with an error marked as "cannot answer", `nar` SHALL print the error and then list every other provider on standard error. Each provider SHALL appear with a number, its default model, and whether its key is set or will be asked for. The user SHALL be able to pick a number, or press Enter to stay.

A pick SHALL switch as `/model <provider>` does: the key is asked for when missing, and the provider is checked. When the switch fails, its error is printed and the list is shown again. After a switch, the same request SHALL run again on the new provider, with the conversation as it was before the failed request. When that request also fails with a "cannot answer" error, the list SHALL be offered again, without the providers already tried.

Enter, end of input or Ctrl+C at the list SHALL keep the previous provider, as before this change. No list SHALL be shown for errors that are not marked, in single mode, or without a terminal.

#### Scenario: Picking another provider
- **WHEN** DeepSeek has no balance, the user asks a question and picks `anthropic` with its key set
- **THEN** the error and the list are shown, Anthropic is checked, the switch is reported, and Anthropic answers the same question

#### Scenario: Staying
- **WHEN** the list is shown and the user presses Enter
- **THEN** nothing switches, the request stays failed, and the prompt comes back

#### Scenario: A pick that fails its check
- **WHEN** the user picks `ollama` and Ollama is not running
- **THEN** the check's error is printed and the list is shown again

#### Scenario: Not offered
- **WHEN** a request fails because the key is rejected, or `nar` runs in single mode or with piped input
- **THEN** only the error is printed, as before
