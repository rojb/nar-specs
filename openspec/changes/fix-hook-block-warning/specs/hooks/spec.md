# Spec Delta

## MODIFIED Requirements

### Requirement: A before_tool hook can refuse the call

A `before_tool` hook refuses the call only when it exits with the block code, which is `2`. On that code the tool SHALL NOT run, and the text the hook wrote to standard output and standard error SHALL reach the model as the result of that call, marked as refused so no failure envelope is added. The user SHALL see that the call was refused by a hook and the text. A refusal is not a failure, so it SHALL NOT produce a warning. A hook that exits with zero SHALL let the call run. A hook that exits with any other code, cannot be started, or exceeds the timeout SHALL let the call run and warn instead, because the runner cannot tell a broken hook from a deliberate refusal and a typo must not lock the project out of its own tools. The warning for that case SHALL name the event, the command, the reason and SHALL say the call was not blocked.

#### Scenario: A hook refuses an edit outside src
- **WHEN** a `before_tool` hook exits with code 2 and prints `edits are only allowed inside src/` while the model calls `edit_file` on `README.md`
- **THEN** the file is not touched and the model receives that sentence as the result of the call

#### Scenario: A hook allows the call
- **WHEN** the `before_tool` hook exits with code 0
- **THEN** the tool runs as usual and its result is unchanged

#### Scenario: A mistyped hook does not lock the project out
- **WHEN** a `before_tool` hook exits with code 1, or its command does not exist and the shell reports that with a non-zero code
- **THEN** the call runs, the model receives the tool's own result, and the user sees a warning that names the command and says it did not block

#### Scenario: No warning on a refusal
- **WHEN** a `before_tool` hook exits with code 2 and prints `blocked by policy`
- **THEN** the call is refused, the user sees that a hook refused it with that text, and no `warning: hook before_tool failed` line is printed

#### Scenario: Hooks run after approval
- **WHEN** the tool needs approval and the user rejects the call
- **THEN** no `before_tool` hook runs for it, because the call will not run
