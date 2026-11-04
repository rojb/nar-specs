# Spec Delta

## Purpose

Defines the human in the loop of `nar`: when the agent asks the user before a tool runs, what the user sees and can answer, what each autonomy level approves on its own, and how rejections and the impossibility of asking are reported back to the model.

## ADDED Requirements

### Requirement: Ask before running a tool that requires approval
Before running a tool that requires approval, `nar` SHALL show the user what the call will do and ask whether it may run. The question SHALL accept three answers: yes (run this call), no (do not run it), and yes for the whole session (run this call and every later call of the same tool without asking again in this session). Answers SHALL be accepted regardless of letter case and in short form (`y`, `n`, `a`) or long form (`yes`, `no`, `always`). An empty or unrecognized answer SHALL NOT run the tool: the question SHALL be asked again. The tool SHALL run only after an explicit yes. Tools that do not require approval SHALL run without any question.

#### Scenario: Approve one call
- **WHEN** the approval level is `ask`, the model calls `shell` with the command `python --version`, and the user answers `y`
- **THEN** the command runs and the model receives its result

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

### Requirement: Preview of the call
The question SHALL be preceded by a preview of what the call will do, produced without changing anything: for a command, the command text and its time limit; for a file edit, the diff of the edit; for a file write, the target path and either a diff against the current content (when the file exists and is text) or the beginning of the new content; for any other tool, the tool name and its arguments. A long preview SHALL be shortened with a note saying how much was left out, and the text of a preview SHALL be shown literally, not interpreted as terminal markup. Showing the preview SHALL NOT modify files, run commands or contact services.

#### Scenario: Diff of an edit
- **WHEN** the model calls `edit_file` replacing `timeout = 10` with `timeout = 30` in `config.py` and approval is needed
- **THEN** before the question the terminal shows a diff that removes `timeout = 10` and adds `timeout = 30`, and `config.py` is unchanged while the question is open

#### Scenario: Command preview
- **WHEN** the model calls `shell` with `command` set to `git status` and `timeout` set to 30, and approval is needed
- **THEN** before the question the terminal shows the command `git status` and the 30 second time limit

#### Scenario: Unknown tool preview
- **WHEN** a tool that provides no preview of its own requires approval and the model calls it with `{"issue": "RRG-1"}`
- **THEN** the question is preceded by the tool name and the argument `issue` with its value

#### Scenario: Very long preview
- **WHEN** the call would write a file of 2,000 lines
- **THEN** the preview shows only the first part, followed by a note stating how many lines were left out, and the question still appears

#### Scenario: Markup-like text is literal
- **WHEN** a previewed diff or command contains text such as `[red]`
- **THEN** the terminal shows that text literally

### Requirement: Autonomy levels
The approval level SHALL decide which calls of tools that require approval are asked about:
- `ask`: every such call is asked about.
- `auto-edit`: calls to tools that change files run without asking when every file they change is inside the working directory where `nar` was started and outside that directory's `.git` and `.nar` folders (which hold hooks and configuration that can later run code); every other such call (commands, changes outside the working directory or inside those folders, any other tool) is asked about.
- `yolo`: no call is asked about and every such call runs.

#### Scenario: Ask level asks for edits
- **WHEN** the approval level is `ask` and the model calls `edit_file` on a file in the working directory
- **THEN** the user is asked before the file is edited

#### Scenario: Auto-edit runs file edits
- **WHEN** the approval level is `auto-edit` and the model calls `edit_file` on `src/app.py` in the working directory
- **THEN** the file is edited without any question and the terminal shows the usual result and diff

#### Scenario: Auto-edit still asks for commands
- **WHEN** the approval level is `auto-edit` and the model calls `shell`
- **THEN** the user is asked before the command runs

#### Scenario: Auto-edit asks for files outside the working directory
- **WHEN** the approval level is `auto-edit` and the model calls `write_file` with a path outside the working directory, such as one reached through `..` or an absolute path elsewhere
- **THEN** the user is asked before the file is written

#### Scenario: Auto-edit asks for configuration and repository internals
- **WHEN** the approval level is `auto-edit` and the model calls `write_file` on `.nar/config.toml` or on `.git/hooks/pre-commit` inside the working directory
- **THEN** the user is asked before the file is written

#### Scenario: Auto-edit asks for unknown tools
- **WHEN** the approval level is `auto-edit` and the model calls a tool that requires approval and is not one of the file tools
- **THEN** the user is asked before it runs

#### Scenario: Yolo never asks
- **WHEN** the approval level is `yolo` and the model calls `shell`, `write_file` or `edit_file`
- **THEN** the tool runs without any question

### Requirement: Approval for the whole session
Answering yes for the whole session SHALL run the current call and SHALL make later calls of the same tool run without a question until the session ends. The memory SHALL be per tool name: it SHALL NOT apply to other tools, SHALL NOT be saved to disk, and SHALL NOT survive a new `nar` process. The answer SHALL be confirmed on the terminal naming the tool. A session-approved tool whose call can never run (a command on the safety list, an edit whose text is not found) SHALL still return its normal error.

#### Scenario: Later calls of the same tool
- **WHEN** the user answers `a` to a call of `edit_file` and the model later calls `edit_file` again in the same interactive session
- **THEN** the second call runs without a question

#### Scenario: Other tools are still asked
- **WHEN** the user answered `a` to `edit_file` and the model then calls `shell`
- **THEN** the user is asked before the command runs

#### Scenario: Not remembered across sessions
- **WHEN** the user answered `a` to `edit_file`, left `nar`, and started it again
- **THEN** the first approval-required call is asked about again

#### Scenario: Session approval covers following requests
- **WHEN** the user answers `a` to `shell` while handling one prompt in interactive mode and then enters a second prompt that makes the model call `shell`
- **THEN** the second call runs without a question

### Requirement: Rejection reaches the model
When the user answers no, the tool SHALL NOT run and the model SHALL receive an error result as the outcome of that call. The result SHALL state that the user rejected the call, SHALL tell the model not to repeat the same call and to propose a different approach or ask the user, and SHALL include the user's reason when one was given. After answering no, the user SHALL be offered to type a reason on one line; pressing Enter without text gives no reason. The request SHALL continue: the model SHALL be called again with the rejection in the conversation. The terminal SHALL show the rejection as a failed call with a short summary.

#### Scenario: Rejection with a reason
- **WHEN** the user answers `n` to an `edit_file` call and types the reason `use the existing helper instead`
- **THEN** the model receives an error result that says the user rejected the call and contains `use the existing helper instead`

#### Scenario: Rejection without a reason
- **WHEN** the user answers `n` and presses Enter at the reason line
- **THEN** the model receives an error result that says the user rejected the call without a reason

#### Scenario: The agent proposes something else
- **WHEN** the approval level is `ask`, the user rejects an `edit_file` call, and the model answers the rejection
- **THEN** the file is unchanged, the request continues, and the model either proposes another change (which is asked about again) or explains what it wanted to do

#### Scenario: Several calls in one answer
- **WHEN** the model requests `write_file` and then `shell` in one answer and the user rejects the first and approves the second
- **THEN** the file is not written, the command runs, and the model receives one result for each call in the original order

### Requirement: Calls that cannot run are not asked about
When the preview shows that a call cannot run (the tool would refuse it without effect), `nar` SHALL NOT ask the user. The model SHALL receive the same error result the tool would have produced. This SHALL apply at every level that would otherwise ask.

#### Scenario: Text to replace is missing
- **WHEN** the approval level is `ask` and the model calls `edit_file` with an `old_string` that is not in the file
- **THEN** no question is shown and the model receives the error saying `old_string` was not found

#### Scenario: Blocked command
- **WHEN** the approval level is `ask` and the model calls `shell` with `rm -rf /`
- **THEN** no question is shown and the model receives the safety list error

### Requirement: Behavior when asking is not possible
When `nar` cannot ask the user, every call that would have been asked about SHALL be refused without running, and the model SHALL receive an error result saying that the tool needs user approval, that approval could not be asked, and how to allow it (the `--approval` option or the `approval` setting). `nar` cannot ask when standard input is not a terminal or when standard error is not a terminal. The refusal SHALL be immediate: `nar` SHALL NOT wait for input, and the same call SHALL get the same answer every time. Calls that need no question at the current level SHALL still run. When input ends while a question is open, the call SHALL be treated as rejected, with no reason, and `nar` SHALL NOT wait again for that call.

#### Scenario: Input is piped
- **WHEN** the user runs `echo "create a.txt" | nar --approval ask "create a.txt"` and the model calls `write_file`
- **THEN** the file is not created, nothing waits for input, and the model receives the error telling it the tool needs user approval that could not be asked, naming `--approval`

#### Scenario: Error stream is redirected
- **WHEN** the user runs `nar "create a.txt" 2> log.txt` at the `ask` level in a terminal and the model calls `write_file`
- **THEN** the file is not created and no question waits for an answer the user cannot see

#### Scenario: Non-interactive run with a level that needs no question
- **WHEN** standard input is not a terminal and the approval level is `yolo`, or the level is `auto-edit` and the model edits a file in the working directory
- **THEN** the tool runs normally

#### Scenario: End of input at the question
- **WHEN** a question is open and the user sends end-of-input (Ctrl+D, or Ctrl+Z then Enter on Windows)
- **THEN** the call is not run, the model receives the rejection without a reason, and `nar` does not hang

### Requirement: Cancelling at the question
Pressing Ctrl+C while a question or the reason line is open SHALL cancel the whole request exactly as Ctrl+C does while the agent is working: the tool does not run, the user is told the request was cancelled, and the request leaves no trace in the conversation.

#### Scenario: Ctrl+C at the question in interactive mode
- **WHEN** the user presses Ctrl+C while the question for a `shell` call is open in interactive mode
- **THEN** the command does not run, `nar` reports the request as cancelled, and the prompt returns

#### Scenario: Ctrl+C at the question in single mode
- **WHEN** the user presses Ctrl+C while the question is open during `nar "..."`
- **THEN** the tool does not run, the request is cancelled without a traceback, and the command exits with code 130

### Requirement: Any tool takes part in approval
The approval behavior SHALL apply to every tool that declares it requires approval, including tools registered after the built-in ones (such as tools provided by external MCP servers), without those tools being named in the approval rules. A tool SHALL be able to provide its own preview and to declare which of its calls only change files inside the working directory; a tool that does neither SHALL be previewed as its name and arguments and SHALL be asked about at every level below `yolo`. An agent built without a way to ask the user (such as one that runs in the background) SHALL refuse approval-required tools as described for the case where asking is not possible.

#### Scenario: Tool added later
- **WHEN** a tool named `jira_create_issue` that requires approval is registered and the model calls it at the `ask` level
- **THEN** the user is asked, with the tool name and arguments as the preview, before it runs

#### Scenario: Tool that provides a preview
- **WHEN** a registered tool provides its own preview text
- **THEN** that text is what the user sees before the question

#### Scenario: Agent without a way to ask
- **WHEN** an agent is built without a way to ask the user and the model calls an approval-required tool at the `ask` level
- **THEN** the tool does not run and the model receives the error saying approval could not be asked
