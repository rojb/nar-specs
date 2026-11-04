# Spec Delta

## ADDED Requirements

### Requirement: Approval level setting
The `approval` setting SHALL select the autonomy level of the agent: `ask` (every call of a tool that requires approval is asked about), `auto-edit` (file edits inside the working directory run on their own; everything else that requires approval is asked about) and `yolo` (nothing is asked). The default SHALL be `ask`. The `--approval` command-line option SHALL override the value from every configuration file for that run, SHALL be validated the same way, and `nar --show-config` SHALL show it with the command line as its source.

#### Scenario: Level from the project file
- **WHEN** `.nar/config.toml` sets `approval = "auto-edit"`
- **THEN** `nar --show-config` shows `approval = auto-edit` with `.nar/config.toml` as its source, and the agent edits files in the working directory without asking

#### Scenario: Flag beats file
- **WHEN** `.nar/config.toml` sets `approval = "ask"` and the user runs `nar --approval yolo --show-config`
- **THEN** the effective approval is `yolo` and its source is shown as the command line

#### Scenario: Default level is the safest
- **WHEN** no configuration file and no option set the approval level
- **THEN** the effective level is `ask` and the agent asks before every call of a tool that requires approval
