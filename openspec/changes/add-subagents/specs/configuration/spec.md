# Spec Delta

## ADDED Requirements

### Requirement: Subagent turn limit
The configuration SHALL support `subagent_max_turns`, the model calls a subagent may make for one task, a whole number of at least 1, default 15.

#### Scenario: Default
- **WHEN** no file sets `subagent_max_turns`
- **THEN** the effective value is 15, and `--show-config` shows it

#### Scenario: Invalid value
- **WHEN** a file sets `subagent_max_turns = 0`
- **THEN** `nar` stops with a one-line error naming the setting and the file, and exit code 1
