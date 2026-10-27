## MODIFIED Requirements

### Requirement: Configuration file locations
`nar` SHALL read settings from a user-level file at `~/.nar/config.toml` and a project-level file at `.nar/config.toml` in the current working directory. Both files are optional and in TOML format. Both files SHALL be read as UTF-8; a UTF-8 byte order mark (BOM) at the start of a file SHALL be ignored, so a file saved with or without a BOM yields the same settings.

#### Scenario: No configuration files
- **WHEN** neither `~/.nar/config.toml` nor `.nar/config.toml` exists
- **THEN** `nar` starts using built-in default values and does not create any file

#### Scenario: Only the user-level file exists
- **WHEN** only `~/.nar/config.toml` exists
- **THEN** its values are used and any missing setting falls back to its default

#### Scenario: File saved with a UTF-8 BOM
- **WHEN** `.nar/config.toml` starts with a UTF-8 BOM followed by `approval = "yolo"` (as written by Windows PowerShell 5.1 `Set-Content -Encoding utf8`)
- **THEN** `nar` starts, and `nar --show-config` reports `approval = yolo` coming from `.nar/config.toml`

#### Scenario: File that is not valid UTF-8
- **WHEN** `.nar/config.toml` contains bytes that are not valid UTF-8
- **THEN** `nar` exits with code 1 and the error names the file
