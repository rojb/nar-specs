# Spec Delta

## ADDED Requirements

### Requirement: Hook timeout setting

`nar` SHALL accept a `hook_timeout` setting: a whole or fractional number of seconds, at least 1 and at most 600, defaulting to `10`. It SHALL be the time each individual hook command gets before it is killed together with its child processes. `--show-config` SHALL show it with the file it came from, like every other setting.

#### Scenario: The default
- **WHEN** no file sets `hook_timeout`
- **THEN** `--show-config` shows `hook_timeout = 10` from the built-in defaults

#### Scenario: A project sets it
- **WHEN** the project file sets `hook_timeout = 30`
- **THEN** every hook gets 30 seconds and `--show-config` shows the project file as its source

#### Scenario: An out-of-range value
- **WHEN** the file sets `hook_timeout = 0` or `hook_timeout = 601`
- **THEN** `nar` refuses to start and the error names `hook_timeout` and the file
