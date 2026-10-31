# shell-tool Specification

## Purpose
Defines the built-in `shell` tool that lets the agent run a command line in the user's terminal shell and read its exit code and output, with a time limit, an output limit and a refusal list for destructive commands.

## Requirements

### Requirement: Run a command
`shell` SHALL accept a required `command` string and run it as one command line in the platform's shell: PowerShell on Windows (PowerShell 7 when installed, otherwise Windows PowerShell) and bash on Linux and macOS (the POSIX `sh` when bash is not installed). The command SHALL run in the working directory where `nar` was started, inherit `nar`'s environment, load no shell profile or startup script, and have its standard input closed so that a command waiting for input ends instead of hanging. The tool SHALL require user approval before running. The tool description given to the model SHALL name the shell in use.

#### Scenario: Run a simple command
- **WHEN** `shell` is called with `command` set to `python --version`
- **THEN** the command runs, and the result reports exit code 0 and the version printed by Python

#### Scenario: Runs in the working directory
- **WHEN** `nar` was started in `/project` and `shell` is called with a command that prints the current folder
- **THEN** the printed folder is `/project`

#### Scenario: Several statements in one call
- **WHEN** `shell` is called with two commands separated by the shell's own separator and both print a line
- **THEN** both lines appear in the output

#### Scenario: Standard input is closed
- **WHEN** `shell` is called with a command that reads from standard input until it ends
- **THEN** the command ends immediately with what it read being empty, and the call does not wait for the timeout

#### Scenario: Command is required
- **WHEN** `shell` is called without `command` or with an empty `command`
- **THEN** the command does not run and the result is an error saying that `command` is required

#### Scenario: Requires approval
- **WHEN** the built-in `shell` tool is inspected
- **THEN** its `requires_approval` is set to true

### Requirement: Command result report
A `shell` call whose command ran to completion SHALL be a success result, whatever the exit code, so the model can use the exit code as information. The result SHALL always state the exit code on its first line, followed by the standard output and the standard error as two separately labelled sections, each present only when the command wrote something to it; when the command wrote nothing the result SHALL say so. Line breaks in the output SHALL be reported as plain line feeds and bytes that are not valid UTF-8 SHALL be replaced instead of causing an error. The terminal summary SHALL be one short line with the exit code and the number of lines the command printed.

#### Scenario: Success with output
- **WHEN** a command prints `Python 3.11.9` and exits with code 0
- **THEN** the result is a success starting with `exit code: 0`, containing a standard output section with `Python 3.11.9` and no standard error section, and the terminal summary is `exit 0 · 1 line`

#### Scenario: Non-zero exit code is not an error
- **WHEN** a command exits with code 3
- **THEN** the result is a success that states `exit code: 3`, and the terminal summary starts with `exit 3`

#### Scenario: Standard error is reported separately
- **WHEN** a command writes `warning` to standard error and `done` to standard output
- **THEN** the result contains a standard output section with `done` and a separate standard error section with `warning`

#### Scenario: No output
- **WHEN** a command exits with code 0 without printing anything
- **THEN** the result states `exit code: 0` and that there was no output, and the terminal summary is `exit 0 · no output`

#### Scenario: Windows line breaks
- **WHEN** a command prints two lines on Windows
- **THEN** the lines in the result are separated by single line feeds, with no carriage returns

#### Scenario: Output that is not valid UTF-8
- **WHEN** a command writes a byte that is not valid UTF-8
- **THEN** the result is still a success, with a replacement character in place of that byte

### Requirement: Timeout
`shell` SHALL accept an optional integer `timeout` in seconds, from 1 to 600, defaulting to 120. When the command is still running after that time the tool SHALL stop it together with every process it started, and return an error result that says the command timed out after that many seconds and was killed, followed by the standard output and standard error captured up to that moment. A timeout outside the allowed range SHALL be rejected as invalid arguments and the command SHALL NOT run.

#### Scenario: Command finishes in time
- **WHEN** `shell` is called with `timeout` set to 10 and a command that ends after one second
- **THEN** the result is a success with the command's exit code

#### Scenario: Command exceeds the timeout
- **WHEN** `shell` is called with `timeout` set to 2 and a command that prints `started` and then waits 30 seconds
- **THEN** the call returns after about 2 seconds with an error result saying the command timed out after 2 seconds and was killed, and the result contains `started`

#### Scenario: Child processes are killed too
- **WHEN** a command that started a child process times out
- **THEN** the child process is no longer running after the call returns

#### Scenario: Default timeout
- **WHEN** `shell` is called without `timeout`
- **THEN** a command that runs longer than 120 seconds is killed and reported as timed out after 120 seconds

#### Scenario: Timeout out of range
- **WHEN** `shell` is called with `timeout` set to 0 or to 601
- **THEN** the command does not run and the error result names `timeout` and its allowed range

### Requirement: Output limit
Each of the standard output and the standard error SHALL be limited to 14,000 characters in the result. When a stream is longer, the result SHALL keep the beginning and the end of that stream and put between them a marker saying how much of that stream was omitted. The limit SHALL hold while the command is running, so a command that prints without stopping cannot exhaust nar's memory. The number of lines in the terminal summary SHALL count every line the command printed, including omitted ones.

#### Scenario: Short output is untouched
- **WHEN** a command prints 100 characters
- **THEN** the result contains all 100 characters and no marker

#### Scenario: Long output keeps both ends
- **WHEN** a command prints 100,000 characters whose first line is `first` and whose last line is `last`
- **THEN** the result contains `first` and `last`, a marker saying that about 86,000 characters were omitted, and no more than 14,000 characters of the command's own output

#### Scenario: Endless output
- **WHEN** a command prints without stopping and the timeout expires
- **THEN** the call returns a timeout error result containing the beginning and the end of what was printed, and nar's memory use stays bounded

#### Scenario: Both streams are limited independently
- **WHEN** a command writes 50,000 characters to standard output and 50,000 characters to standard error
- **THEN** each section is limited to 14,000 characters and the whole result stays under the 30,000-character tool output limit

### Requirement: Dangerous command blocklist
Before running anything, `shell` SHALL check the command against a list of destructive command patterns and, on a match, SHALL NOT run it and SHALL return an error result that says the command was blocked by the safety list, which kind of command it matched and that the user has to run it themselves. The check SHALL ignore letter case and the amount of whitespace, SHALL apply the same list whichever shell is in use, and SHALL cover at least: recursive removal of the filesystem root, a drive root or the user's home folder (`rm -rf /`, `rm -rf ~`, `Remove-Item -Recurse -Force C:\`, `rd /s /q C:\`, `del /s /q C:\`); formatting or creating a filesystem on a disk (`format C:`, `mkfs.ext4 /dev/sda1`, `Format-Volume`); writing to a disk device (`dd if=/dev/zero of=/dev/sda`, `> /dev/sda`); a fork bomb (`:(){ :|:& };:`); and shutting down or rebooting the machine (`shutdown`, `reboot`, `Stop-Computer`, `Restart-Computer`). Commands that only mention these words as text, or that act on a project path, SHALL NOT be blocked. The list is a safety net against mistakes and SHALL NOT be presented as a sandbox; approval remains the control that decides whether a command runs.

#### Scenario: Recursive delete of the root
- **WHEN** `shell` is called with `rm -rf /`
- **THEN** the command does not run and the result is an error saying it was blocked by the safety list

#### Scenario: Variations are still blocked
- **WHEN** `shell` is called with `RM   -fr   ~/` or with `sudo rm -rf /*`
- **THEN** the command does not run and the result is an error saying it was blocked

#### Scenario: Windows destructive commands
- **WHEN** `shell` is called with `Remove-Item -Recurse -Force C:\` or `del /s /q C:\`
- **THEN** the command does not run and the result is an error saying it was blocked

#### Scenario: Formatting a disk
- **WHEN** `shell` is called with `format C:` or `mkfs.ext4 /dev/sda1`
- **THEN** the command does not run and the result is an error saying it was blocked

#### Scenario: Fork bomb
- **WHEN** `shell` is called with `:(){ :|:& };:`
- **THEN** the command does not run and the result is an error saying it was blocked

#### Scenario: Shutdown
- **WHEN** `shell` is called with `shutdown /s /t 0` or `sudo reboot`
- **THEN** the command does not run and the result is an error saying it was blocked

#### Scenario: Blocked in a chained command
- **WHEN** `shell` is called with `echo done; rm -rf /`
- **THEN** the command does not run at all, including the `echo`

#### Scenario: Harmless commands are not blocked
- **WHEN** `shell` is called with `rm -rf build/`, `rm -rf ./node_modules`, `echo "do not shutdown"` or `git log --format=%H`
- **THEN** the command is not blocked and runs

### Requirement: Cancellation
When the user cancels the request with Ctrl+C while a `shell` command is running, the tool SHALL stop the command and every process it started before the cancellation continues, so that nothing keeps running in the background after the request is cancelled.

#### Scenario: Ctrl+C during a command
- **WHEN** a command that waits 60 seconds is running and the user presses Ctrl+C
- **THEN** the command and any process it started are killed, and nar returns to its prompt as for any cancelled request

### Requirement: Shell errors
`shell` SHALL return an error result, never crash, when the command cannot be started: when no suitable shell is installed, when the working directory no longer exists, or when the operating system refuses to start the process. The message SHALL say what failed.

#### Scenario: No shell available
- **WHEN** neither PowerShell nor bash nor sh can be found
- **THEN** `shell` returns an error result saying that no shell was found

#### Scenario: Working directory removed
- **WHEN** the working directory where `nar` was started no longer exists and `shell` is called
- **THEN** the result is an error naming the missing folder and no command runs

#### Scenario: Unknown program inside the command
- **WHEN** `shell` is called with a command that names a program that does not exist
- **THEN** the result is a success that carries the shell's own error message in the standard error section and a non-zero exit code

### Requirement: Shell tool registration
`shell` SHALL be registered as a built-in tool so that it appears in the provider-neutral export, can be offered to the model and can be run by the developer tool runner.

#### Scenario: Listed in the export
- **WHEN** the registry of built-in tools is exported
- **THEN** the export contains an object named `shell` with a description and a parameters schema that requires `command` and describes `timeout` as an integer from 1 to 600

#### Scenario: Agent runs it only when approval allows
- **WHEN** the approval level is not `yolo` and the model calls `shell`
- **THEN** the command does not run and the model receives the error result about needing user approval

#### Scenario: Agent uses the output
- **WHEN** the approval level is `yolo` and the user asks the agent which Python version is installed
- **THEN** the agent calls `shell` with a command such as `python --version` and its answer states the version printed by the command
