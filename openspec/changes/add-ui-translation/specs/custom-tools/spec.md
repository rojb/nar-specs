## MODIFIED Requirements

### Requirement: Confirmation for project tools
`nar` SHALL NOT run any file from `.nar/tools/` until the user has allowed that folder with those exact contents. Allowed means one of: a remembered answer whose folder and fingerprint of file names and contents match, a "yes" to the question (`y`, `yes`, `s`, `si` or `sí`, in any letter case), or `--trust-project-tools` on the command line. The question SHALL be asked on standard error, SHALL list the files, and SHALL say that they run as code with the user's permissions. A "yes" SHALL be remembered in `trusted_tools.json` in the nar folder, and so SHALL the flag. A "no" (`n` or `no`), end of input or Ctrl+C at the question SHALL skip the project tools for this run, with a line saying so, without remembering anything and without ending `nar`. No setting in a configuration file and no approval level SHALL allow project tools.

#### Scenario: First time
- **WHEN** the user runs `nar` in a project with `.nar/tools/a.py` and answers yes
- **THEN** the tool is available, and the next run in that project does not ask

#### Scenario: Allowed in Spanish
- **WHEN** the interface language is English and the user answers `sí` to the question
- **THEN** the tool files are allowed and remembered, exactly as for `y`

#### Scenario: Refused
- **WHEN** the user answers no
- **THEN** the file is not run, its tools are not available, and `nar` continues; the next run asks again

#### Scenario: A file changes
- **WHEN** a previously allowed project adds a tool file or changes one
- **THEN** `nar` asks again before running any of them

#### Scenario: Cannot ask
- **WHEN** standard input or standard error is not a terminal and the project was not allowed
- **THEN** the files are not run, a warning names `--trust-project-tools`, and the run continues with exit code 0

#### Scenario: Configuration cannot allow it
- **WHEN** the project's `.nar/config.toml` sets `approval = "yolo"` and the project was not allowed
- **THEN** `nar` still asks, and it does not run the files until the user answers yes

#### Scenario: Global tools never ask
- **WHEN** only the global folder has tool files
- **THEN** no question is asked, also without a terminal
