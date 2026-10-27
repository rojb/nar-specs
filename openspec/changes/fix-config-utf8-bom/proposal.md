# Proposal

- **Owner:** Ruben
- **Linear issue:** RRG-80
- **Video timestamps covered:** none (bug found while preparing a manual test session on Windows).
- **Depends on:** RRG-60 `scaffold-cli` (archived), which owns the `configuration` capability.
- **Course requirement covered:** none.

## Why

On Windows, `nar` refuses to start when `.nar/config.toml` or `~/.nar/config.toml` begins with a UTF-8 byte order mark (BOM): it exits with `invalid TOML in ... (line 1): Invalid statement`. Windows PowerShell 5.1 (`Set-Content -Encoding utf8`, `Out-File`) and older Notepad versions write UTF-8 with a BOM, so a user who creates the file with the default tools of their system gets an error that points at a line that looks correct. Observed on 2026-10-27 while preparing a test folder with PowerShell 5.1.

## What Changes

- Configuration files are read as UTF-8 and a leading BOM is ignored, so a file saved with or without a BOM gives the same settings.
- A file that is not valid UTF-8 keeps failing with a clear "cannot read" error that names the file; malformed TOML keeps failing with the file and line, as today.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `configuration`: the "Configuration file locations" requirement states the file encoding (UTF-8, with or without BOM) and gains a scenario for a file that starts with a BOM.

## Impact

- `nar` repo: `src/nar/config/loader.py` (`_read_toml`) and `tests/test_config.py`.
- No new dependency, no new config key, no change to other settings or errors.
