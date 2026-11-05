# Spec Delta

## MODIFIED Requirements

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
