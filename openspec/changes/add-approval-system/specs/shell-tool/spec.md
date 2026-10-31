# Spec Delta

## ADDED Requirements

### Requirement: Shell preview
The description of a `shell` call made before it runs SHALL show the exact command text, the shell that would run it and the time limit that would apply, and SHALL NOT start any process. When running the call would be refused without effect (empty command, command on the safety list, no shell found), the description SHALL carry the same error message that running it would return. A `shell` call SHALL NOT be declared as a project edit, because a command can do anything.

#### Scenario: Preview of a command
- **WHEN** `shell` is described for `git status` without `timeout`
- **THEN** the description shows `git status`, the name of the shell in use and the default time limit of 120 seconds, and no process is started

#### Scenario: Preview of a blocked command
- **WHEN** `shell` is described for `rm -rf /`
- **THEN** the description carries the safety list error, and no process is started

#### Scenario: Not a project edit
- **WHEN** `shell` is asked whether a call only edits project files
- **THEN** the answer is no, whatever the command

## MODIFIED Requirements

### Requirement: Shell tool registration
`shell` SHALL be registered as a built-in tool so that it appears in the provider-neutral export, can be offered to the model and can be run by the developer tool runner.

#### Scenario: Listed in the export
- **WHEN** the registry of built-in tools is exported
- **THEN** the export contains an object named `shell` with a description and a parameters schema that requires `command` and describes `timeout` as an integer from 1 to 600

#### Scenario: Agent runs it only when approval allows
- **WHEN** the approval level is `ask` or `auto-edit` and the model calls `shell`
- **THEN** the user sees the command and is asked before it runs; if the user rejects it, the command does not run and the model receives the rejection as an error result

#### Scenario: Agent uses the output
- **WHEN** the approval level is `yolo` and the user asks the agent which Python version is installed
- **THEN** the agent calls `shell` with a command such as `python --version` and its answer states the version printed by the command
