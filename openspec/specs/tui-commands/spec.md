# tui-commands Specification

## Purpose
Defines the interactive command surface of `nar`: the router that runs a `/` command, the completion
offered at the prompt, the commands every session has, and what the commands that change the session
in place do to the conversation.

## Requirements

### Requirement: Slash commands are listed and completed

In interactive mode, typed text starting with `/` SHALL be a command and SHALL be routed to the command registered under that name. Entering `/help` SHALL list every registered command as `/name` followed by its one-line summary, in alphabetical order. When standard input is a terminal, the prompt SHALL complete command names as the user types a `/` command and SHALL show the summary of the highlighted one. When standard input is not a terminal (a pipe or a redirected file), the prompt SHALL read a plain line without completion and everything else SHALL behave the same.

#### Scenario: Help lists everything registered
- **WHEN** the user types `/help`
- **THEN** every registered command appears with its summary, including the ones other changes added (`/mcp`, `/compact`, `/sessions`, `/resume`)

#### Scenario: Completion
- **WHEN** the user types `/co` at a terminal prompt
- **THEN** `/compact` and `/config` are offered with their summaries

#### Scenario: A line that is not a command
- **WHEN** the user types `hello /help` or `/etc/hosts is what`
- **THEN** the line is sent to the model as a request, not treated as a command

#### Scenario: Piped input
- **WHEN** the input is piped (`printf '/help\n' | nar`)
- **THEN** the command runs and no completion machinery is loaded

#### Scenario: An unknown command
- **WHEN** the user types `/nope`
- **THEN** an error lists the registered commands and the session continues

### Requirement: The base commands

Every session SHALL have `/help`, `/clear`, `/exit`, `/model`, `/approval`, `/config`, `/tools` and `/tokens`, in addition to the commands other changes register. `exit` without a slash SHALL keep working as it does today. No command SHALL write its output to standard output: it goes to standard error as literal text, so a redirected run keeps only the answer.

#### Scenario: Exit
- **WHEN** the user types `/exit`
- **THEN** the session ends with exit code 0, exactly like `exit`

#### Scenario: A command that takes no argument
- **WHEN** the user types `/help extra`
- **THEN** the command runs and the argument is ignored, except where a command documents one

#### Scenario: A command registered by another change answers on standard error
- **WHEN** the user types `/mcp`, `/sessions` or `/checkpoints`
- **THEN** the answer appears on standard error and standard output holds only the banner and the prompt

#### Scenario: A redirected interactive run keeps only the answers
- **WHEN** `nar` reads commands from standard input and standard output is redirected to a file
- **THEN** the file holds only the banner, the prompt and the model's answers, never a command's answer

### Requirement: /model changes provider and model in place

`/model` with no argument SHALL show the provider and model in use, with the source they came from. `/model <provider>` SHALL switch to that provider's default model, `/model <provider>/<model>` and `/model <provider> <model>` SHALL switch to both, and an unknown provider SHALL be rejected with the list of providers without changing anything. A switch SHALL keep the conversation, the tools and the session, SHALL apply to the next model call, and SHALL report the new provider and model. A switch to a provider whose credentials are missing SHALL fail with a one-line error, and the previous provider SHALL stay in use.

#### Scenario: Switching providers keeps the conversation
- **WHEN** the user asks a question, types `/model ollama`, and asks a follow-up
- **THEN** the session stays, the follow-up is answered by Ollama, and it can refer to the earlier answer

#### Scenario: Missing credentials
- **WHEN** the user types `/model anthropic` without an Anthropic key
- **THEN** a one-line error says the variable is missing and the next request still uses the previous provider

#### Scenario: Showing the current one
- **WHEN** the user types `/model`
- **THEN** the provider, the model and the source are shown and nothing changes

### Requirement: /approval changes the autonomy level in place

`/approval` with no argument SHALL show the level in use. `/approval ask|auto-edit|yolo` SHALL change it for the following tool calls in the same session, and an unknown level SHALL be rejected with the three names without changing anything.

#### Scenario: From ask to yolo
- **WHEN** the user types `/approval yolo` and then asks for a shell command
- **THEN** the command runs without asking

### Requirement: /clear starts a new conversation

`/clear` SHALL replace the conversation with the system prompt alone, SHALL say that the earlier messages are gone, and SHALL keep the session, the provider and the approval level. Later requests SHALL NOT contain any earlier message. The session file SHALL NOT keep the discarded messages.

#### Scenario: Clearing
- **WHEN** the user asks something, types `/clear`, and asks again
- **THEN** the second request is sent with no earlier message and the session continues

### Requirement: Inspecting the session

`/config` SHALL show the effective configuration with the file each value came from, hiding secrets the way `--show-config` does. `/tools` SHALL list every registered tool name with its one-line description, including the MCP tools of the connected servers. `/tokens` SHALL show the tokens used, the context window and the percentage, like the line shown after each request.

#### Scenario: Secrets stay hidden
- **WHEN** a server header holds a literal secret and the user types `/config`
- **THEN** the value is shown masked

#### Scenario: MCP tools are listed
- **WHEN** an MCP server is connected and the user types `/tools`
- **THEN** its `mcp__<server>__<tool>` names appear with the built-in tools
