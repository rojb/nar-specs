# subagents Specification

## Purpose
Defines the `task` tool: subagents with their own conversation that return only a final answer, the `explorer` and `general` types, their limits, how approvals and hooks apply to them, and how their activity is shown. The research behind it is in `docs/SUBAGENTES.md`.

## Requirements

### Requirement: Task tool
The model SHALL be able to call a `task` tool with a subagent type and a prompt. `nar` SHALL run a subagent with its own new conversation, holding only that prompt, and with its own system prompt, which SHALL include the project's `AGENTS.md` when one was loaded. The subagent SHALL NOT see the main conversation. The call itself SHALL NOT ask for approval. Several `task` calls in one turn SHALL run one after the other.

#### Scenario: A subagent does not see the conversation
- **WHEN** the main conversation holds earlier messages and the model calls `task`
- **THEN** the first model call of the subagent contains only the prompt given to `task`

#### Scenario: Unknown type
- **WHEN** the model calls `task` with a type that does not exist
- **THEN** it receives an error result naming the valid types, and no subagent runs

### Requirement: Isolated context
The main conversation SHALL grow only with the `task` call and its result. The subagent's messages SHALL NOT be added to the main conversation or to the saved session. The subagent's conversation SHALL be managed with the same context limit as the main one, so it prunes and compacts on its own.

#### Scenario: Main context grows only with the summary
- **WHEN** a subagent reads ten large files and answers with a short summary
- **THEN** the main conversation has one tool call and one result holding that summary, and none of the file contents

#### Scenario: Session file
- **WHEN** a session is saved after a request that used `task`
- **THEN** the file holds the call and the summary, and not the subagent's messages

### Requirement: Subagent types
The `explorer` type SHALL have only `read_file`, `grep`, `glob` and `list_dir`. The `general` type SHALL have every tool of the main agent at the time of the call, including custom and MCP tools, except `task` and `todos`. The tools a subagent does not have SHALL NOT be offered to its model, and a call to one SHALL be an unknown tool error.

#### Scenario: Explorer cannot write
- **WHEN** an `explorer`'s model asks for `write_file` or `shell`
- **THEN** it receives an unknown tool error and nothing is changed

#### Scenario: General has MCP and custom tools
- **WHEN** MCP servers or custom tools are available and a `general` subagent runs
- **THEN** its model is offered those tools

#### Scenario: The plan stays with the main agent
- **WHEN** the main agent has a `todos` list and a `general` subagent runs
- **THEN** the subagent is not offered `todos`, and the list does not change

### Requirement: Limits
A subagent SHALL stop after `subagent_max_turns` model calls. Subagents SHALL NOT launch subagents: `task` SHALL NOT be available to them, so depth is at most one.

#### Scenario: Nesting
- **WHEN** a subagent's model asks for `task`
- **THEN** it receives an unknown tool error

#### Scenario: Turn limit
- **WHEN** a subagent reaches its turn limit
- **THEN** the result holds its last text and says the work may be incomplete, and the main agent continues

### Requirement: Approvals and hooks in a subagent
A subagent's tool calls SHALL go through the same approval level, the same "always allow" answers and the same hooks as the main agent's. The waiting indicator SHALL stop before the user is asked. Where `nar` cannot ask, a call that needs approval SHALL be refused, and the subagent SHALL be told, as in the main agent.

#### Scenario: General asks before writing
- **WHEN** the level is `ask` and a `general` subagent calls `write_file`
- **THEN** the user is asked, and on `n` the file is not changed and the subagent is told

#### Scenario: Always allow is shared
- **WHEN** the user answered `a` for `edit_file` in the main agent, and a subagent then calls `edit_file`
- **THEN** it does not ask again

#### Scenario: A hook blocks a subagent's call
- **WHEN** a `before_tool` hook blocks `shell`, and a `general` subagent calls `shell`
- **THEN** the command does not run and the subagent is told

### Requirement: Result of a subagent
The result of `task` SHALL be the text of the subagent's last answer, within the usual limit on tool output. A provider error, a stop for repeated calls, or a subagent ending with no text SHALL be an error result or carry a note saying what happened, and the main agent SHALL continue. Ctrl+C SHALL cancel the whole request.

#### Scenario: Provider error
- **WHEN** the provider fails during a subagent
- **THEN** the main agent receives an error result naming the failure, and it can answer or try something else

#### Scenario: Empty answer
- **WHEN** a subagent ends without any text
- **THEN** the main agent receives an error result saying so

#### Scenario: Cancelled
- **WHEN** the user presses Ctrl+C during a subagent
- **THEN** the whole request is cancelled, and the main conversation has no trace of it

### Requirement: Folded activity
While a subagent works, the user SHALL see on standard error one updating line with its type and number of tool calls, on a terminal. When it ends, the terminal SHALL show the `task` call and one result line with the type, tool calls and turns. The subagent's answer text and tool activity SHALL NOT be printed in full, and nothing from it SHALL reach standard output directly.

#### Scenario: Output stays clean
- **WHEN** the user runs `nar "explain the architecture" > answer.txt` and the model uses `task`
- **THEN** the file holds only the main agent's answer

### Requirement: Delegation guidance
The main agent's system prompt SHALL tell the model to use `task` with `explorer` for broad searches of the code, and not for reading a known file.

#### Scenario: Exploring a repository
- **WHEN** the user asks "explore this repo and explain the architecture"
- **THEN** the system prompt and the tool description lead the model to launch an `explorer`

### Requirement: Subagent reply language
The system prompt of every subagent SHALL tell it to reply in the configured language, with the same exceptions as the main system prompt: code, identifiers, file names, paths, commands, tool arguments and quoted text stay unchanged.

#### Scenario: Subagent told to reply in Spanish
- **WHEN** the language is `es` and the model calls `task`
- **THEN** the first model call of the subagent has a system prompt that tells it to reply in Spanish

#### Scenario: Subagent told to reply in English
- **WHEN** the language is `en` and the model calls `task`
- **THEN** the first model call of the subagent has a system prompt that tells it to reply in English
