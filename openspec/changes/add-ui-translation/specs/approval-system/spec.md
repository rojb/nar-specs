## MODIFIED Requirements

### Requirement: Ask before running a tool that requires approval
Before running a tool that requires approval, `nar` SHALL show the user what the call will do and ask whether it may run. The question SHALL accept three answers: yes (run this call), no (do not run it), and yes for the whole session (run this call and every later call of the same tool without asking again in this session). Answers SHALL be accepted regardless of letter case and in short form (`y`, `n`, `a`) or long form (`yes`, `no`, `always`), and also in Spanish (`s`, `si`, `sí`, `siempre`), whatever the interface language is; accents SHALL NOT matter. An empty or unrecognized answer SHALL NOT run the tool: the question SHALL be asked again. The tool SHALL run only after an explicit yes. Tools that do not require approval SHALL run without any question.

#### Scenario: Approve one call
- **WHEN** the approval level is `ask`, the model calls `shell` with the command `python --version`, and the user answers `y`
- **THEN** the command runs and the model receives its result

#### Scenario: Approve in Spanish
- **WHEN** the approval level is `ask`, the interface language is English, the model calls `shell`, and the user answers `Sí`
- **THEN** the command runs, exactly as if the user had answered `y`

#### Scenario: Always in Spanish
- **WHEN** the interface language is Spanish and the user answers `siempre` to a call of `edit_file`
- **THEN** the call runs and later calls of `edit_file` in the session run without a question

#### Scenario: Reject one call
- **WHEN** the approval level is `ask`, the model calls `write_file`, and the user answers `n`
- **THEN** the file is not written and the model receives an error result saying the user rejected the call

#### Scenario: Empty answer does not approve
- **WHEN** the question is shown and the user presses Enter without typing anything
- **THEN** nothing runs, the question is shown again, and it keeps being shown until the user gives a recognized answer

#### Scenario: Read-only tools are not asked
- **WHEN** the approval level is `ask` and the model calls `read_file`
- **THEN** the tool runs without any question

#### Scenario: Question comes after the call line
- **WHEN** the model calls an approval-required tool in a terminal
- **THEN** the terminal shows the usual line with the tool name and arguments, then the preview of the call, then the question, and the waiting indicator is not shown while the question is open

#### Scenario: Question does not pollute the answer stream
- **WHEN** the user runs `nar "..." > answer.txt` in a terminal and the agent asks for approval
- **THEN** the preview and the question appear on the terminal (standard error) and `answer.txt` contains only the model's text
