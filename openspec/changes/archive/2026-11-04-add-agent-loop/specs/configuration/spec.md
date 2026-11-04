# Spec Delta

## ADDED Requirements

### Requirement: Turn limit setting
The configuration SHALL support a `max_turns` setting: the maximum number of model calls the agent makes for one request. It SHALL be a whole number of at least 1, with default 20.

#### Scenario: Default turn limit
- **WHEN** no configuration file sets `max_turns`
- **THEN** the effective `max_turns` is 20 and `nar --show-config` shows it with source `default`

#### Scenario: Invalid turn limit
- **WHEN** a configuration file sets `max_turns = 0`
- **THEN** `nar` exits with code 1 and the error names the `max_turns` setting and the file
