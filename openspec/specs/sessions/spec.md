# sessions Specification

## Purpose
TBD - created by archiving change add-sessions. Update Purpose after archive.

## Requirements

### Requirement: Saved sessions
Each run of `nar` that sends at least one request SHALL have a session, saved to `sessions/<id>.json` in the nar folder (`~/.nar`, or `NAR_HOME`). A session SHALL hold its id, creation and update dates, working folder, provider, model and every message of the conversation. It SHALL be saved before each model call and when each request ends. The file SHALL be replaced atomically, so an interrupted write leaves the previous version complete. On Linux and macOS only the user SHALL be able to read it. A run with no request SHALL NOT create a file.

#### Scenario: Saved after a request
- **WHEN** the user sends a request in interactive mode and it finishes
- **THEN** a session file exists with the request and the answer

#### Scenario: Terminal closed during a request
- **WHEN** the process dies while a tool is running
- **THEN** the file holds the conversation up to the last model call, every tool call in it has a result, and a provider accepts it after resuming

#### Scenario: Cancelled request
- **WHEN** the user cancels a request with Ctrl+C
- **THEN** the saved session has no trace of it, and if it was the first request no session file remains

#### Scenario: Reduced conversation
- **WHEN** the conversation was pruned or compacted
- **THEN** the saved messages are the reduced ones

#### Scenario: Nothing sent
- **WHEN** the user starts `nar` and types `exit`
- **THEN** no session file is created

### Requirement: Resuming from the command line
`nar --resume` SHALL continue the most recently updated session of the current working folder. `nar --resume <id>` SHALL continue that session; a beginning of the id that matches exactly one session SHALL be enough. A prompt given as well SHALL run in single mode and continue the session. A session started in another folder SHALL be resumed with a warning naming that folder, and tools SHALL run in the current folder. The system prompt SHALL be built for the current run, and the provider and model SHALL come from the current configuration, with a note when they differ from the saved ones. "Always allow" answers SHALL NOT be restored.

#### Scenario: Continue where it stopped
- **WHEN** the user closes the terminal, runs `nar --resume` in the same folder and asks about something from the earlier conversation
- **THEN** the model receives the earlier messages and answers from them, and new messages are saved to the same session

#### Scenario: No session in this folder
- **WHEN** the user runs `nar --resume` in a folder without sessions
- **THEN** `nar` stops with a one-line error and exit code 1 before contacting any provider or MCP server

#### Scenario: Unknown or ambiguous id
- **WHEN** the id matches no session, or a beginning matches more than one
- **THEN** `nar` stops with a one-line error saying so, naming the matches when there are several, and exit code 1

#### Scenario: Corrupt session file
- **WHEN** the file to resume is not a valid session
- **THEN** `nar` stops with a one-line error naming the file, leaves the file untouched, and exit code 1

#### Scenario: Other folder
- **WHEN** the user resumes by id a session started in another folder
- **THEN** a warning names that folder and the session continues in the current one

### Requirement: Earlier history is shown
When a session is resumed in interactive mode, `nar` SHALL show its earlier user requests, assistant answers and tool calls on standard error, literally, without running anything again and without tool results. It SHALL show at most the last 20 messages, saying how many were left out, and SHALL cut each one to 2,000 characters. Single mode SHALL show only a line naming the session and its number of messages.

#### Scenario: History on resume
- **WHEN** a session with two requests is resumed in interactive mode
- **THEN** both requests and answers are shown before the prompt, and nothing is added to standard output

#### Scenario: Text that looks like markup
- **WHEN** a saved message contains `[red]x[/red]`
- **THEN** it is shown as written

### Requirement: Session commands
In interactive mode, `/sessions` SHALL list saved sessions, most recently updated first and at most 20, with id, update date, number of messages, folder, the beginning of the first request, and a mark on the current one. Files that are not valid sessions SHALL be left out with a warning saying how many. `/resume <id>` SHALL save the current session and switch to the given one, showing its history. `/resume` without an id SHALL do so with the latest other session of the current folder. A failure SHALL print an error and leave the current session active.

#### Scenario: List
- **WHEN** the user types `/sessions`
- **THEN** the sessions are listed without calling the model

#### Scenario: Switch
- **WHEN** the user types `/resume <id>` and then a request
- **THEN** the model receives that session's messages, and the request is saved in that session and not in the previous one

#### Scenario: Nothing to resume
- **WHEN** the user types `/resume` and the folder has no other session
- **THEN** an error says so and the current session continues

### Requirement: Save failures
A session that cannot be saved SHALL NOT stop the request or the session. `nar` SHALL warn once per run, and SHALL warn again only after a save has succeeded and a later one fails.

#### Scenario: Folder not writable
- **WHEN** the sessions folder cannot be written
- **THEN** one warning is shown and the requests continue with exit code 0
