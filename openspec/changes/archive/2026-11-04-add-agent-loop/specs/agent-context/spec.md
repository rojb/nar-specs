# Spec Delta

## Purpose

Defines the conversation the agent keeps: what the system prompt tells the model, how the history grows across requests, how it stays consistent when a request is cancelled or fails, and how its size in tokens is measured.

## ADDED Requirements

### Requirement: System prompt
Every model call SHALL include a system prompt that tells the model it is `nar`, a coding agent working in the user's project, how to use its tools, and the current working directory, operating system and date. When project instructions from `AGENTS.md` were loaded, the system prompt SHALL include them, marked as the project's instructions.

#### Scenario: Project instructions included
- **WHEN** `nar` starts in a project whose `AGENTS.md` says "Always answer in Spanish"
- **THEN** the system prompt sent to the model contains that text under a heading that identifies it as project instructions

#### Scenario: No project instructions
- **WHEN** no `AGENTS.md` was found
- **THEN** the system prompt is sent without a project-instructions section

#### Scenario: Environment details
- **WHEN** `nar` starts in `/home/user/project` on Linux
- **THEN** the system prompt names that directory and the operating system

### Requirement: Conversation history
The conversation SHALL keep, in order, every user request, every assistant answer (with its tool calls) and every tool result of the session. In interactive mode, each new request SHALL be sent together with the earlier ones, so the model can refer to them.

#### Scenario: Follow-up question
- **WHEN** in interactive mode the user asks "read pyproject.toml" and then "what version did it say?"
- **THEN** the second request is answered using the file contents from the first, without reading the file again

### Requirement: Consistent history after an interrupted request
A request that is cancelled, fails, or stops at the turn limit SHALL NOT leave the conversation in a state the providers reject. A cancelled or failed request SHALL be removed from the conversation entirely. A request stopped at the turn limit SHALL keep its completed work, and every tool call in the history SHALL have a result.

#### Scenario: Turn limit keeps a valid history
- **WHEN** a request stops at the turn limit with unexecuted tool calls in the last answer, and the user sends another request
- **THEN** each of those tool calls has an error result in the history saying it was not run, and the provider accepts the next request

### Requirement: Token count
The conversation SHALL expose its current size in tokens. When the provider reported usage for the last call, the count SHALL be based on it plus an estimate for anything added afterwards; otherwise it SHALL be an estimate of the whole conversation, including the system prompt.

#### Scenario: Count grows with the conversation
- **WHEN** a request adds a long tool result to the conversation
- **THEN** the token count afterwards is larger than before the request
