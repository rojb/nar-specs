# Spec Delta

## MODIFIED Requirements

### Requirement: A before_tool hook can refuse the call

A `before_tool` hook refuses the call only when it exits with the block code, which is `2`. On that code the tool SHALL NOT run, and the text the hook wrote to standard output and standard error SHALL reach the model as the result of that call, marked as refused so no failure envelope is added. The user SHALL see that the call was refused by a hook and the text. A hook that exits with zero SHALL let the call run. A hook that exits with any other code, cannot be started, or exceeds the timeout SHALL let the call run and warn instead, because the runner cannot tell a broken hook from a deliberate refusal and a typo must not lock the project out of its own tools. The warning for that case SHALL name the event, the command, the reason and SHALL say the call was not blocked.

#### Scenario: A hook refuses an edit outside src
- **WHEN** a `before_tool` hook exits with code 2 and prints `edits are only allowed inside src/` while the model calls `edit_file` on `README.md`
- **THEN** the file is not touched and the model receives that sentence as the result of the call

#### Scenario: A hook allows the call
- **WHEN** the `before_tool` hook exits with code 0
- **THEN** the tool runs as usual and its result is unchanged

#### Scenario: A mistyped hook does not lock the project out
- **WHEN** a `before_tool` hook exits with code 1, or its command does not exist and the shell reports that with a non-zero code
- **THEN** the call runs, the model receives the tool's own result, and the user sees a warning that names the command and says it did not block

#### Scenario: Hooks run after approval
- **WHEN** the tool needs approval and the user rejects the call
- **THEN** no `before_tool` hook runs for it, because the call will not run

### Requirement: A failing hook never breaks the request

A hook that cannot be started, exits with a non-zero code on an event that cannot block, exits with a code that is not the block code on `before_tool`, or exceeds the hook timeout SHALL produce a warning on standard error and the request SHALL continue. A `before_tool` hook that cannot be started, times out or exits with any code other than the block code SHALL NOT block the call. The warning SHALL name the event and the command, SHALL say whether the call was blocked, and SHALL NOT print more than a documented number of characters of the hook's output.

#### Scenario: A missing command
- **WHEN** the configured command does not exist
- **THEN** one warning names it, the request continues, and a `before_tool` hook of that command does not block the call

#### Scenario: A hook that hangs
- **WHEN** a hook runs longer than the hook timeout
- **THEN** it is killed together with its child processes, a warning says it timed out, and the request continues

#### Scenario: An after_tool hook that fails
- **WHEN** an `after_tool` hook exits with code 1
- **THEN** the tool's result reaches the model unchanged and the user sees a warning
