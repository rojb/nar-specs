# hooks Specification

## Purpose
Defines how a project makes `nar` run its own shell commands around the agent loop: which events
exist, what each hook receives, how a `before_tool` hook can refuse a call, and how a broken hook is
kept from breaking the agent.

## Requirements

### Requirement: Hooks are configured as shell commands

`nar` SHALL read a `[hooks]` table from the project configuration and from the user configuration, with one key per event: `before_agent`, `before_tool`, `after_tool`, `after_agent` and `on_error`. Each value SHALL be a list of command strings, run in that order. An unknown key in `[hooks]` SHALL be a configuration error naming the key and the file, like any other unknown setting. With no `[hooks]` table `nar` SHALL start no process, import no hook module and behave exactly as it does without this capability.

#### Scenario: A project defines two hooks
- **WHEN** the project file has `[hooks]` with `after_tool = ["ruff check ."]` and `on_error = ["notify.sh"]`
- **THEN** the two hooks run for their events and no other event runs anything

#### Scenario: A typo in the table
- **WHEN** the file has `[hooks]` with `before_tolls = ["x"]`
- **THEN** `nar` refuses to start and the error names `hooks.before_tolls` and the file

#### Scenario: No hooks configured
- **WHEN** neither file has a `[hooks]` table
- **THEN** the agent loop behaves exactly as before, with no extra process

### Requirement: Each event carries its own JSON on standard input

Every hook SHALL receive one JSON object on standard input, ending with a newline, whose `event` field names the event. `before_agent` SHALL carry the user's `prompt`; `after_agent` SHALL carry the `prompt`, the `outcome` (`finished`, `turn_limit`, `loop`, `cancelled` or `error`) and the number of `turns`; `before_tool` and `after_tool` SHALL carry the `tool` name and its `arguments`; `after_tool` SHALL also carry `is_error` and the tool's `output`, cut to a documented length; `on_error` SHALL carry `where` (a tool or the provider) and the `message`. Every payload SHALL also carry the `cwd`.

#### Scenario: The before_tool payload
- **WHEN** the model calls `edit_file` with `{"path": "src/app.py"}` and a `before_tool` hook is configured
- **THEN** the hook reads a JSON object with `event` equal to `before_tool`, `tool` equal to `edit_file`, the arguments and the working directory

#### Scenario: The after_agent payload
- **WHEN** a request ends because the turn limit was reached and an `after_agent` hook is configured
- **THEN** the hook reads a JSON object whose `outcome` is `turn_limit` and whose `turns` is the limit

### Requirement: A before_tool hook can refuse the call

A `before_tool` hook whose exit code is not zero SHALL stop the call: the tool SHALL NOT run, and the text the hook wrote to standard output and standard error SHALL reach the model as the result of that call, marked as refused so no failure envelope is added. The user SHALL see that the call was refused by a hook and the text. A hook that exits with zero SHALL let the call run.

#### Scenario: A hook refuses an edit outside src
- **WHEN** a `before_tool` hook exits with code 1 and prints `edits are only allowed inside src/` while the model calls `edit_file` on `README.md`
- **THEN** the file is not touched and the model receives that sentence as the result of the call

#### Scenario: A hook allows the call
- **WHEN** the `before_tool` hook exits with code 0
- **THEN** the tool runs as usual and its result is unchanged

#### Scenario: Hooks run after approval
- **WHEN** the tool needs approval and the user rejects the call
- **THEN** no `before_tool` hook runs for it, because the call will not run

### Requirement: A failing hook never breaks the request

A hook that cannot be started, exits with a non-zero code on an event that cannot block, or exceeds the hook timeout SHALL produce a warning on standard error and the request SHALL continue. A `before_tool` hook that cannot be started or times out SHALL NOT block the call. The warning SHALL name the event and the command, and SHALL NOT print more than a documented number of characters of the hook's output.

#### Scenario: A missing command
- **WHEN** the configured command does not exist
- **THEN** one warning names it, the request continues, and a `before_tool` hook of that command does not block the call

#### Scenario: A hook that hangs
- **WHEN** a hook runs longer than the hook timeout
- **THEN** it is killed together with its child processes, a warning says it timed out, and the request continues

#### Scenario: An after_tool hook that fails
- **WHEN** an `after_tool` hook exits with code 1
- **THEN** the tool's result reaches the model unchanged and the user sees a warning

### Requirement: Hook output is visible to the user

What a hook prints SHALL be shown to the user on standard error, as literal text cut to a documented length, and SHALL NOT be written to standard output. The model SHALL NOT receive the output of a hook except when a `before_tool` hook refuses the call.

#### Scenario: The formatter says nothing
- **WHEN** a hook exits with code 0 and prints nothing
- **THEN** nothing extra is shown

#### Scenario: The linter prints a finding
- **WHEN** an `after_tool` hook prints two lines on standard output
- **THEN** those two lines appear on standard error and standard output keeps only the answer

### Requirement: Hook events follow the request

`before_agent` SHALL run once before the first model call of a request, `after_agent` SHALL run once when the request ends for any reason (an answer, the turn limit, a loop, a cancellation or an error), `on_error` SHALL run for each failing tool result and for a provider failure, and `before_tool` and `after_tool` SHALL run around each executed call, after its approval and before the next model call. A cancellation SHALL still run `after_agent`.

#### Scenario: A request cancelled with Ctrl+C
- **WHEN** the user presses Ctrl+C during a request and an `after_agent` hook is configured
- **THEN** the hook runs with `outcome` equal to `cancelled`
