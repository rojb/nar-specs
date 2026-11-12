# Spec Delta

## MODIFIED Requirements

### Requirement: System prompt
Every model call SHALL include a system prompt that tells the model it is `nar`, a coding agent working in the user's project, how to use its tools, and the current working directory, operating system and date. When project instructions from `AGENTS.md` were loaded, the system prompt SHALL include them, marked as the project's instructions. When at least one skill is available, the system prompt SHALL list every available skill by name and description, in name order, and SHALL tell the model to call the `skill` tool before working on a request that matches a skill's description. The system prompt SHALL NOT include the instructions of any skill.

#### Scenario: Project instructions included
- **WHEN** `nar` starts in a project whose `AGENTS.md` says "Always answer in Spanish"
- **THEN** the system prompt sent to the model contains that text under a heading that identifies it as project instructions

#### Scenario: No project instructions
- **WHEN** no `AGENTS.md` was found
- **THEN** the system prompt is sent without a project-instructions section

#### Scenario: Environment details
- **WHEN** `nar` starts in `/home/user/project` on Linux
- **THEN** the system prompt names that directory and the operating system

#### Scenario: Skills listed
- **WHEN** the skills `demo` and `release` are available
- **THEN** the system prompt lists both with their descriptions and says to use the `skill` tool, and it does not contain the instructions of either skill

#### Scenario: No skills
- **WHEN** no skill is available
- **THEN** the system prompt has no skills section
