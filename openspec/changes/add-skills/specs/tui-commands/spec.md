# Spec Delta

## ADDED Requirements

### Requirement: /skills lists the available skills

`/skills` SHALL list every available skill on standard error, in name order, one per line, with its name, its description, and whether it came from the project or the user. When no skill is available, `/skills` SHALL say so and name the two folders where skills are looked for.

#### Scenario: Listing
- **WHEN** the project skill `demo` and the user skill `release` are available and the user types `/skills`
- **THEN** both appear with their descriptions, `demo` marked as project and `release` marked as user

#### Scenario: Nothing found
- **WHEN** no skill is available and the user types `/skills`
- **THEN** a line says no skills were found and names `.nar/skills/` and the user skills folder

### Requirement: /skill uses a skill from the prompt

`/skill <name>` SHALL add the instructions of that skill to the next request sent to the model, marked as the skill the user chose, and SHALL say on standard error that the skill will be used. `/skill <name> <request>` SHALL send `<request>` right away, with the skill's instructions added in the same way. The instructions SHALL be read from disk when the command runs. `/skill` with no name, or with a name that is not available, SHALL print an error that lists the available skills, and nothing SHALL be added to the conversation.

#### Scenario: Skill for the next request
- **WHEN** the user types `/skill demo` and then `write the changelog`
- **THEN** the request sent to the model holds the instructions of `demo` and the text `write the changelog`

#### Scenario: Skill and request together
- **WHEN** the user types `/skill demo write the changelog`
- **THEN** the request is sent at once, with the instructions of `demo` and the text `write the changelog`

#### Scenario: Unknown skill
- **WHEN** the user types `/skill nope`
- **THEN** an error lists the available skills and the next request is sent without any skill

#### Scenario: Cleared before use
- **WHEN** the user types `/skill demo` and then `/clear`
- **THEN** the next request is sent without the instructions of `demo`
