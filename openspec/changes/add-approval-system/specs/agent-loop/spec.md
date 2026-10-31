# Spec Delta

## MODIFIED Requirements

### Requirement: Tools that require approval
A tool marked as requiring approval SHALL run only as the approval system allows: it SHALL be asked about, auto-approved or refused according to the approval level and the user's answers. A call that is not allowed to run SHALL NOT run, and the model SHALL receive an error result for it so the request continues. The loop SHALL keep returning one result per requested call, in order, whatever the outcome of each approval.

#### Scenario: Approval-required tool at level ask
- **WHEN** the approval level is `ask`, the model calls a tool that requires approval, and the user rejects the call
- **THEN** the tool does not run and the model receives an error result saying the user rejected it

#### Scenario: Approval-required tool at level yolo
- **WHEN** the approval level is `yolo` and the model calls a tool that requires approval
- **THEN** the tool runs normally without any question

#### Scenario: Approval-required tool when nobody can be asked
- **WHEN** the approval level is `ask`, standard input is not a terminal, and the model calls a tool that requires approval
- **THEN** the tool does not run and the model receives an error result explaining that the tool needs user approval that could not be asked
