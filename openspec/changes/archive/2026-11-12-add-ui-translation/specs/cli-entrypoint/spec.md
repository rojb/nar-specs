## MODIFIED Requirements

### Requirement: Help output
The `nar` command SHALL print usage information describing its modes and options when invoked with `--help`. The usage text, the option descriptions and Click's own words SHALL be shown in the interface language, worked out before the configuration is loaded.

#### Scenario: Help flag
- **WHEN** the user runs `nar --help`
- **THEN** the output describes how to start interactive mode and single mode, lists the available options, and the command exits with code 0

#### Scenario: Help in Spanish by default
- **WHEN** the user runs `nar --help` with no configuration
- **THEN** the usage text and the option descriptions are in Spanish

#### Scenario: Help in English
- **WHEN** the user runs `nar --language en --help`
- **THEN** the usage text and the option descriptions are in English
