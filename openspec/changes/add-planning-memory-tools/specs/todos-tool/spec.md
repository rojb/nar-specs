# Spec Delta

## Purpose

Defines the built-in `todos` tool that lets the agent keep a visible task list for the current session: how the list is written, the allowed statuses, and how it is shown to the user.

## ADDED Requirements

### Requirement: Write the task list
`todos` SHALL accept a required `todos` array where each item has a non-empty `content` and a `status` of `pending`, `in_progress` or `completed`. Each call SHALL replace the whole list for the rest of the session. An empty array SHALL clear the list. It SHALL NOT require user approval.

#### Scenario: Create a plan
- **WHEN** `todos` is called with three items: "Read config loader" `in_progress`, "Add max_turns" `pending`, "Update README" `pending`
- **THEN** the result is a success and the session's list holds those three items in that order

#### Scenario: Update progress
- **WHEN** the list holds three items and `todos` is called again with the first item `completed` and the second `in_progress`
- **THEN** the session's list holds the new statuses

#### Scenario: Invalid status
- **WHEN** an item has `status` `done`
- **THEN** the tool does not run and the error result names the `status` field and the allowed values

### Requirement: One task in progress
At most one item SHALL have status `in_progress`. A list with more than one SHALL produce an error result and SHALL leave the previous list unchanged.

#### Scenario: Two tasks in progress
- **WHEN** `todos` is called with two items marked `in_progress`
- **THEN** the result is an error saying only one task can be in progress, and the previous list is kept

### Requirement: Checklist shown to the user and the model
After every successful call, the user SHALL see the whole list as a checklist below the tool's success line, one line per item in order, with a marker for each status: `[ ]` pending, `[>]` in progress, `[x]` completed. The model SHALL receive the same checklist and counts of completed and remaining tasks.

#### Scenario: Checklist display
- **WHEN** the list is "Read config loader" completed, "Add max_turns" in progress, "Update README" pending
- **THEN** the terminal shows `[x] Read config loader`, `[>] Add max_turns` and `[ ] Update README` on separate lines below the success line

#### Scenario: Cleared list
- **WHEN** `todos` is called with an empty array
- **THEN** the result is a success saying the task list is empty

### Requirement: List scope
The task list SHALL belong to the current `nar` process: it SHALL persist between requests of the same interactive session and SHALL start empty in every new session.

#### Scenario: Next request in the same session
- **WHEN** in interactive mode the agent writes a list during one request and the user sends another request
- **THEN** the list written earlier is still the session's list until the agent writes a new one
