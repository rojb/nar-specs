# Spec Delta

## Purpose

Defines how `nar` finds skills (folders with a `SKILL.md` file of specialized instructions), what a valid skill looks like, which one wins when two share a name, and how the model reads a skill's instructions only when it needs them.

## ADDED Requirements

### Requirement: Skill discovery

At startup, in interactive and single mode, `nar` SHALL look for skills in `skills/` of the nar folder (`~/.nar`, or `NAR_HOME` when set) and in `.nar/skills/` of the working directory. A skill SHALL be a folder directly inside one of those folders that contains a file named `SKILL.md`. Folders without `SKILL.md`, files directly inside the skills folder, and anything deeper than one level SHALL be ignored without a warning. A missing skills folder SHALL NOT produce a warning.

#### Scenario: Project skill
- **WHEN** `nar` starts in a directory that has `.nar/skills/demo/SKILL.md`
- **THEN** the skill `demo` is available, marked as coming from the project

#### Scenario: User skill
- **WHEN** `~/.nar/skills/release/SKILL.md` exists and `NAR_HOME` is not set
- **THEN** the skill `release` is available in every project, marked as coming from the user

#### Scenario: NAR_HOME
- **WHEN** `NAR_HOME` is set to `/tmp/narhome` and `/tmp/narhome/skills/demo/SKILL.md` exists
- **THEN** `demo` is available and `~/.nar/skills/` is not read

#### Scenario: No skills
- **WHEN** neither skills folder exists
- **THEN** `nar` starts normally, with no warning and no skills

### Requirement: SKILL.md format

A `SKILL.md` file SHALL start with a YAML frontmatter block delimited by lines containing only `---`, followed by the skill's instructions in Markdown. The frontmatter SHALL have a `name` and a `description`, both non-empty strings. `name` SHALL contain only lowercase letters, digits and hyphens, SHALL be at most 64 characters long, and SHALL be equal to the name of the skill's folder. `description` SHALL be at most 1024 characters long. Other frontmatter keys SHALL be ignored. The skill's folder MAY contain other files that the instructions refer to.

#### Scenario: Valid skill
- **WHEN** `.nar/skills/demo/SKILL.md` has frontmatter `name: demo` and `description: Shows how skills work`, followed by instructions
- **THEN** `demo` is available with that description

#### Scenario: Extra keys
- **WHEN** the frontmatter also has `license: MIT`
- **THEN** the skill loads and the extra key has no effect

### Requirement: Broken skills are skipped

A skill that cannot be used SHALL produce one warning, naming its `SKILL.md` file and the reason, and SHALL be skipped. The other skills SHALL still load and `nar` SHALL still start. This SHALL cover a file that cannot be read or is not valid UTF-8, a missing or unclosed frontmatter block, frontmatter that is not valid YAML or not a mapping, a missing or empty `name` or `description`, a `name` that breaks the format rules or does not match its folder, and a `description` that is too long. Warnings SHALL go to standard error.

#### Scenario: No frontmatter
- **WHEN** `.nar/skills/broken/SKILL.md` contains only Markdown
- **THEN** one warning names that file and says the frontmatter is missing, and the other skills load

#### Scenario: Name does not match the folder
- **WHEN** `.nar/skills/demo/SKILL.md` has `name: other`
- **THEN** one warning names that file and says the name does not match the folder, and the skill is not available

#### Scenario: Unreadable file
- **WHEN** a `SKILL.md` cannot be read
- **THEN** one warning names the file and `nar` starts normally

### Requirement: Project skills take precedence

When the user folder and the project folder both have a skill with the same name, the project skill SHALL be the one available, and the user skill SHALL be ignored without a warning. Skills SHALL be listed in name order.

#### Scenario: Same name in both places
- **WHEN** both `~/.nar/skills/demo/SKILL.md` and `.nar/skills/demo/SKILL.md` are valid
- **THEN** only the project's `demo` is available, and using it returns the project's instructions

### Requirement: The skill tool

When at least one skill is available, `nar` SHALL register a tool named `skill` that takes the name of a skill and returns that skill's instructions (the text after the frontmatter) together with the absolute path of the skill's folder, so the model can read the files it mentions. The tool SHALL NOT require approval. The instructions SHALL be read from disk when the tool is called. When no skill is available, the `skill` tool SHALL NOT be registered.

#### Scenario: Using a skill
- **WHEN** the model calls `skill` with `name: demo`
- **THEN** the result holds the instructions of `demo` and the path of its folder, and no approval is asked

#### Scenario: Unknown skill
- **WHEN** the model calls `skill` with a name that is not available
- **THEN** the result is an error that lists the available skill names

#### Scenario: File changed or removed after startup
- **WHEN** a skill's `SKILL.md` is deleted or becomes unreadable after startup and the model calls `skill` with its name
- **THEN** the result is an error naming the file, and the session continues

#### Scenario: No skills
- **WHEN** no skill is available
- **THEN** `/tools` does not list `skill`

### Requirement: Skill files are never run by nar

`nar` SHALL treat everything in a skill folder as text. It SHALL NOT run any file from a skill folder by itself, and loading project skills SHALL NOT ask the user to allow the folder. A command or script that a skill's instructions mention SHALL run only when the model calls a tool for it, and SHALL go through the approval system like any other call.

#### Scenario: Skill that mentions a script
- **WHEN** a skill's folder has `run.sh` and its instructions say to run it
- **THEN** nothing runs at startup or when the skill is used, and running `run.sh` only happens through a shell tool call that follows the approval level in use
