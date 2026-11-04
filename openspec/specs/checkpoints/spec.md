# checkpoints Specification

## Purpose
Defines checkpoints: points inside a session that the conversation can return to with `/checkpoint`, `/checkpoints` and `/restore`. A session is the whole conversation; a checkpoint is one point in it. Restoring discards later messages and does not restore files.

## Requirements

### Requirement: Saving a checkpoint
In interactive mode, `/checkpoint [name]` SHALL save the current conversation as a checkpoint of the current session, without calling the model, and SHALL show its number, its name if it has one, and how many messages it holds. Numbers SHALL start at 1 in each session and SHALL never be reused. A name SHALL be at most 60 characters, SHALL NOT be only digits, and SHALL be unique in the session; anything else SHALL be an error that saves nothing. An empty conversation SHALL produce an error that there is nothing to save.

#### Scenario: Named checkpoint
- **WHEN** the user types `/checkpoint before refactor` after a request
- **THEN** `nar` shows `checkpoint 1 saved: before refactor` with the number of messages, and the prompt comes back

#### Scenario: Empty conversation
- **WHEN** the user types `/checkpoint` before any request
- **THEN** an error says there is nothing to save, and no file is created

#### Scenario: Repeated name
- **WHEN** the session already has a checkpoint named `base` and the user types `/checkpoint base`
- **THEN** an error says that name is taken and nothing is saved

### Requirement: Listing checkpoints
In interactive mode, `/checkpoints` SHALL list the checkpoints of the current session in the order they were saved, with number, date, number of messages and name, without calling the model. A session without checkpoints SHALL say so.

#### Scenario: List
- **WHEN** the session has two checkpoints and the user types `/checkpoints`
- **THEN** both are listed, oldest first

#### Scenario: Only this session
- **WHEN** another session has checkpoints and the current one has none
- **THEN** `/checkpoints` says this session has none

### Requirement: Restoring a checkpoint
In interactive mode, `/restore <id or name>` SHALL replace the conversation with the messages of that checkpoint, without calling the model, discarding later messages, and SHALL save the session. The next request SHALL be sent with those messages only. `nar` SHALL say how many messages the conversation now has and that files changed by tools were not restored. Later checkpoints SHALL remain available.

#### Scenario: Later messages are discarded
- **WHEN** the user saves a checkpoint, makes another request, restores the checkpoint and asks about that request
- **THEN** the model receives only the messages up to the checkpoint plus the new question, and the session file no longer holds the discarded request

#### Scenario: Undoing a restore
- **WHEN** the user saves checkpoints 1 and 2, restores 1, and then restores 2
- **THEN** the conversation is the one saved in checkpoint 2

#### Scenario: Files stay as they are
- **WHEN** a tool changed a file after the checkpoint and the user restores it
- **THEN** the file is unchanged and `nar` says files were not restored

#### Scenario: Reduced conversation
- **WHEN** a checkpoint was saved before a compaction and the user restores it after the compaction
- **THEN** the conversation has the messages it had when the checkpoint was saved

#### Scenario: Unknown checkpoint
- **WHEN** the user types `/restore 9` or `/restore nope` and it does not exist, or `/restore` with nothing
- **THEN** an error says so, and the conversation is unchanged

### Requirement: Checkpoints last with the session
Checkpoints SHALL be saved in a file of their own for each session, replaced atomically and, on Linux and macOS, readable only by the user. They SHALL be available after closing `nar` and resuming the session, and SHALL follow the current session after `/resume`. They SHALL NOT appear as sessions in `/sessions`.

#### Scenario: After resuming
- **WHEN** the user saves a checkpoint, leaves, runs `nar --resume` and types `/checkpoints`
- **THEN** the checkpoint is listed and can be restored

#### Scenario: Not a session
- **WHEN** a session has checkpoints and the user types `/sessions`
- **THEN** no extra entry or invalid-file warning appears

### Requirement: Checkpoint failures
A checkpoint that cannot be written or read SHALL print an error and SHALL NOT change the conversation or end the session. A checkpoint file that is not valid SHALL NOT be overwritten.

#### Scenario: Corrupt file
- **WHEN** the session's checkpoint file is not valid and the user types `/checkpoint`
- **THEN** an error names the file, the file is untouched, and the session continues

#### Scenario: Folder not writable
- **WHEN** the checkpoint cannot be written
- **THEN** an error says so and the session continues
