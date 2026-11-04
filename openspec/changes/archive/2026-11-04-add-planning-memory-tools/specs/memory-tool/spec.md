# Spec Delta

## Purpose

Defines the built-in `memory` tool that lets the agent keep short facts about the user and the project across sessions: saving, deleting and listing entries, where they are stored, their limits, and how saved memory reaches the agent in a new session.

## ADDED Requirements

### Requirement: Save, delete and list entries
`memory` SHALL accept a required `action` of `save`, `delete` or `list`. `save` SHALL store a `value` under a `key`, replacing any value already stored under that key. `delete` SHALL remove the entry with that `key`. `list` SHALL return every entry as `key: value` lines in key order. It SHALL NOT require user approval.

#### Scenario: Save a preference
- **WHEN** `memory` is called with `action` `save`, `key` `language` and `value` `Answer in Spanish`
- **THEN** the result is a success confirming that `language` was saved, and a later `list` shows `language: Answer in Spanish`

#### Scenario: Replace a value
- **WHEN** `language` is saved with `Answer in Spanish` and then saved again with `Answer in English`
- **THEN** `list` shows only `language: Answer in English`

#### Scenario: Delete an entry
- **WHEN** `memory` is called with `action` `delete` and `key` `language`
- **THEN** the entry is removed and `list` no longer shows it

#### Scenario: Missing arguments
- **WHEN** `save` is called without `value`, or `delete` without `key`
- **THEN** the result is an error naming the missing argument and nothing is stored or removed

#### Scenario: Unknown key
- **WHEN** `delete` is called with a key that is not stored
- **THEN** the result is an error saying the key does not exist and listing the stored keys

### Requirement: Storage between sessions
Entries SHALL be stored in `memory.json` inside the user-level `nar` folder (`~/.nar`, or `$NAR_HOME` when set), created on the first save, and SHALL be available to every later `nar` process. A write SHALL never leave a partially written file.

#### Scenario: New process sees saved entries
- **WHEN** one `nar` process saves `test_runner: pytest` and later a new `nar` process lists the memory
- **THEN** the new process lists `test_runner: pytest`

#### Scenario: Nothing saved yet
- **WHEN** `list` is called and no memory file exists
- **THEN** the result is a success saying the memory is empty, and no file is created

### Requirement: Limits
Keys SHALL be 1 to 64 characters of letters, digits, `_`, `-` and `.`. Values SHALL be 1 to 500 characters. At most 50 entries SHALL be stored. Saving beyond a limit SHALL produce an error result explaining the limit and SHALL leave the stored memory unchanged.

#### Scenario: Value too long
- **WHEN** `save` is called with a 2,000-character `value`
- **THEN** the result is an error saying values are limited to 500 characters, and nothing is stored

#### Scenario: Memory full
- **WHEN** 50 entries are stored and `save` is called with a new key
- **THEN** the result is an error saying the memory is full and suggesting deleting entries; replacing an existing key still works

### Requirement: Unreadable memory file
When `memory.json` exists but is not valid memory data, `nar` SHALL NOT overwrite it: the tool SHALL return an error naming the file, and a session SHALL start with a warning and without saved memory.

#### Scenario: Corrupt file
- **WHEN** `memory.json` contains `not json` and the agent calls `save`
- **THEN** the result is an error naming the file and saying it could not be read, and the file content is unchanged

### Requirement: Saved memory at session start
At the start of every session (single or interactive mode), the saved entries SHALL be included in the agent's system prompt under a heading that identifies them as memory saved in earlier sessions. With no entries, no memory section SHALL be added.

#### Scenario: Preference remembered in a new session
- **WHEN** in one `nar` run the agent saved `language: Answer in Spanish`, and later the user runs `nar "what do you remember about my preferences?"`
- **THEN** the system prompt of the new run contains `language: Answer in Spanish` and the answer mentions that preference
