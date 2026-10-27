# Spec Delta

## Purpose

Defines how `nar` turns a user request into a sequence of model calls and tool calls, when that sequence stops, and how the agent's work is shown to the user.

## ADDED Requirements

### Requirement: Agentic loop
For each user request, `nar` SHALL send the conversation and the definitions of the registered tools to the selected model. When the model's answer requests tool calls, `nar` SHALL run each requested tool, add the results to the conversation and ask the model again. The request SHALL end when the model answers without requesting any tool.

#### Scenario: Answer that needs a tool
- **WHEN** the user asks "read pyproject.toml and tell me the version" in a project whose `pyproject.toml` declares a version
- **THEN** the model calls `read_file` on `pyproject.toml`, receives its contents, and its final answer states that version

#### Scenario: Answer without tools
- **WHEN** the user asks "say hello"
- **THEN** the model answers directly, no tool runs, and the request ends after one model call

### Requirement: Tool calls are executed in order
When one model answer requests several tool calls, `nar` SHALL run them one after another in the order the model gave and SHALL return a result for every one of them before calling the model again. Unknown tools, invalid arguments and failing tools SHALL produce error results for the model and SHALL NOT stop the request.

#### Scenario: Two calls in one answer
- **WHEN** the model requests `read_file` on `a.txt` and on `b.txt` in the same answer
- **THEN** both files are read, in that order, and the next model call receives both results

#### Scenario: Tool error goes back to the model
- **WHEN** the model calls `read_file` on a file that does not exist
- **THEN** the model receives an error result naming the missing file and can continue, and the request is not aborted

### Requirement: Turn limit
A turn is one call to the model. A single request SHALL make at most `max_turns` turns. When the limit is reached while the model is still requesting tools, the request SHALL stop and the user SHALL be told that the turn limit was reached and how to raise it.

#### Scenario: Limit reached
- **WHEN** `max_turns = 2` and the model requests a tool in each of its first two answers
- **THEN** exactly two model calls are made, the tool calls of the second answer are not run, and the user sees a message naming the limit of 2 and the `max_turns` setting

### Requirement: Tools that require approval
Until approvals exist, a tool marked as requiring approval SHALL run only when the approval level is `yolo`. At any other level the tool SHALL NOT run and the model SHALL receive an error result saying the tool needs user approval, which is not available yet.

#### Scenario: Approval-required tool at level ask
- **WHEN** the approval level is `ask` and the model calls a tool that requires approval
- **THEN** the tool does not run and the model receives an error result explaining that the tool needs user approval

#### Scenario: Approval-required tool at level yolo
- **WHEN** the approval level is `yolo` and the model calls a tool that requires approval
- **THEN** the tool runs normally

### Requirement: Streamed and visible activity
The model's text SHALL be shown while it is generated. Each tool call and its outcome SHALL be shown as defined by the tool activity display. While waiting for the model's first output of a turn, the user SHALL see that `nar` is working. The answer text SHALL go to standard output; tool activity, progress indicators and warnings SHALL go to standard error.

#### Scenario: Output separation
- **WHEN** the user runs `nar "read pyproject.toml and tell me the version" > answer.txt`
- **THEN** `answer.txt` contains only the model's text, while the `read_file` call and its result are shown on the terminal

#### Scenario: Waiting indicator
- **WHEN** the model takes several seconds before producing its first text or tool call
- **THEN** an indicator on standard error shows that `nar` is waiting for the model, and it disappears when output starts

### Requirement: Cancelling a request
Pressing Ctrl+C while a request is running SHALL stop it: the current model call or tool SHALL be abandoned, the user SHALL be told the request was cancelled, and the request SHALL leave no trace in the conversation.

#### Scenario: Cancel during a model call
- **WHEN** the user presses Ctrl+C while the model is streaming an answer in interactive mode
- **THEN** streaming stops, a cancellation notice is shown, the prompt returns, and the next request is sent without the cancelled one in the conversation

### Requirement: Provider failure during a request
When the model cannot be reached or rejects a call during a request (after the provider's own retries), the request SHALL stop, the provider's one-line error SHALL be shown, and the request SHALL leave no trace in the conversation.

#### Scenario: Failure in interactive mode
- **WHEN** a provider error happens during a request in interactive mode
- **THEN** the error is shown on one line, the session continues, and the prompt returns
